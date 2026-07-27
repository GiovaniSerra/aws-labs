# AWS Hands-On Lab: Managing Access at Scale with Amazon S3 Access Points

## Overview
This laboratory project demonstrates how to manage data access policies at scale using **Amazon S3 Access Points**. 

A medical imaging company stores sensitive patient records (dental files, prescriptions, and X-ray images) in Amazon S3. To adhere to the principle of least privilege, specific IAM identities need restricted access based on data category and network origin.

Using S3 Access Points, bucket policies, and VPC endpoints, access control was delegated to dedicated endpoints with strict tag-based conditions and network perimeter controls.

---

## Why Use S3 Access Points?

Managing access permissions using a single S3 Bucket Policy works well for small environments. However, as data lakes grow and dozens of applications require distinct permissions, a centralized bucket policy quickly becomes complex, error-prone, and reaches character limits.

S3 Access Points solve this by decentralizing access management, creating dedicated entry points tailored to specific workloads, IAM identities, or network paths.

| Feature | S3 Bucket Policy | S3 Access Point |
| :--- | :--- | :--- |
| **Governance Model** | Centralized (Single monolithic policy) | Decentralized (Multiple dedicated policies) |
| **Maintainability** | Difficult to scale and audit | Highly scalable and modular |
| **Application Isolation** | Shared single policy for all apps | One unique endpoint per application or team |
| **Network Control** | Broad bucket-level network rules | Granular VPC and network-origin restrictions |

---

## Architecture Evolution: Folder Prefixes vs. Access Points

### Traditional Model (Prefix-Based Control)
Historically, data segregation relied on folder paths inside a shared bucket managed by a single policy:

```text
medical-records-bucket/
├── dental/  ---> (Shared bucket policy managing dental access)
└── xray/    ---> (Shared bucket policy managing xray access)
```

Modern Model (Access Point + ABAC + VPC)
With S3 Access Points, governance is decoupled. Access is enforced at dedicated entry points with Attribute-Based Access Control (ABAC) and network boundaries:

```
medical-records-bucket
   │
   ├──> [ dental-ap ] ──> ABAC (recordtype=dental) ──> VPC Private Access
   │
   └──> [ xray-ap   ] ──> ABAC (recordtype=xray)   ──> VPC Private Access
```
---

### Architecture & Interview Deep-Dive — Scope of S3 Access Points

Global Bucket Names vs. Regional Endpoints: While S3 bucket names share a globally unique namespace across all AWS accounts, S3 Access Points are regional resources. Each Access Point is created within a specific AWS Region and Account ID, and its Amazon Resource Name (ARN) explicitly includes the region (e.g., arn:aws:s3:us-east-1:123456789012:accesspoint/dental-ap).

Independent DNS Hostnames: Every Access Point automatically receives its own unique, region-specific DNS alias (e.g., dental-ap-123456789012-s3alias.s3-accesspoint.us-east-1.amazonaws.com). This allows applications to target the endpoint directly without relying on standard bucket naming rules.

Cross-Region Architecture Note: Because Access Points are bound to a single region, cross-region access requires using S3 Multi-Region Access Points (MRAPs), which leverage AWS Global Accelerator to route client traffic to the nearest regional bucket replica over the AWS global network.

---

### Why Gateway Endpoint instead of Interface Endpoint?

For Amazon S3 traffic originating inside a VPC, a **Gateway VPC Endpoint** was selected over an Interface Endpoint (PrivateLink) for two main architectural reasons:

| Criteria | Gateway VPC Endpoint | Interface VPC Endpoint (PrivateLink) |
| :--- | :--- | :--- |
| **Cost** | **Free** (No hourly charges or data processing fees) | Billed per endpoint hour + data processing fees |
| **Routing** | Uses VPC Route Tables (Direct gateway target) | Uses Elastic Network Interfaces (ENIs) with Private IPs |
| **Best Use Case** | Bulk/Standard S3 access within the same region | On-premises connectivity or cross-region VPC peering |

---

## Architecture Diagram

![Architecture Diagram](https://github.com/GiovaniSerra/aws-labs/blob/main/s3-access-points-lab/images/ar.jfif)

## Key Objectives
* **Scale Access Management**: Explain and implement Amazon S3 Access Points to decentralize permission management.
* **Implement ABAC Rules**: Create custom Access Point policies using Attribute-Based Access Control (`s3:ExistingObjectTag`).
* **Enforce Network Isolation**: Restrict S3 traffic through a Gateway VPC Endpoint for private cloud network isolation.
* **Delegate Access Control**: Configure an S3 Bucket Policy to delegate access control directly to S3 Access Points.

---

## Step-by-Step Implementation
Task 1: Environment Assessment
IAM Identities: Inspected DentalUser and XrayUser. Neither identity had direct attached policies or group memberships.

S3 Data Classification: Inspected the pre-populated S3 bucket (medical-records-*). Objects are classified using tags:

Dental files: recordtype = dental

X-ray files: recordtype = xray

Bucket Protection: Confirmed that Block Public Access is fully active across the bucket.

### Task 2: Provision S3 Access Points
Created two dedicated S3 Access Points attached to the Lab VPC to enforce fine-grained access control based on object tags (recordtype).

1. Dental Access Point (dental-ap)
Configured an S3 Access Point allowing DentalUser to list and retrieve objects tagged with recordtype: dental.

Name: dental-ap

Network Origin: Virtual Private Cloud (VPC)

VPC ID: <LAB_VPC_ID>

Block Public Access: Enabled

![Dental Access Point Policy](https://github.com/GiovaniSerra/aws-labs/blob/main/s3-access-points-lab/images/ap%20policy%20-%20dental-ap.png)

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::<ACCOUNT_ID>:user/DentalUser"
            },
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:<REGION>:<ACCOUNT_ID>:accesspoint/dental-ap"
        },
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::<ACCOUNT_ID>:user/DentalUser"
            },
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:<REGION>:<ACCOUNT_ID>:accesspoint/dental-ap/object/*",
            "Condition": {
                "StringEquals": {
                    "s3:ExistingObjectTag/recordtype": "dental"
                }
            }
        }
    ]
}
```
X-ray Access Point (xray-ap)
Configured a dedicated Access Point allowing XrayUser to list and retrieve objects tagged with recordtype: xray.

```JSON
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::<ACCOUNT_ID>:user/XrayUser"
            },
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:<REGION>:<ACCOUNT_ID>:accesspoint/xray-ap"
        },
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::<ACCOUNT_ID>:user/XrayUser"
            },
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:<REGION>:<ACCOUNT_ID>:accesspoint/xray-ap/object/*",
            "Condition": {
                "StringEquals": {
                    "s3:ExistingObjectTag/recordtype": "xray"
                }
            }
        }
    ]
}
```

Access Points Management Dashboard
Confirmed both Access Points are active and bound exclusively to the private Virtual Private Cloud (VPC).

S3 Console view of active Access Points (dental-ap and xray-ap). Both are bound to a Virtual Private Cloud (VPC) network origin, blocking external internet entry.

Task 3: Create Gateway VPC Endpoint & Network Perimeter
Provisioned a Gateway VPC Endpoint (vpce-*) to route S3 traffic internally within the AWS network without exposing data to the public internet.

Active Gateway VPC Endpoint configured in the AWS VPC Console, ensuring all S3 calls remain within the private network perimeter.

![VPC Gateway Endpoint Details](https://github.com/GiovaniSerra/aws-labs/blob/main/s3-access-points-lab/images/vpc.png)

---

### Task 4: Delegate Bucket Access Control
Applied a resource policy to the primary S3 bucket (`medical-records-*`), delegating control directly to any Access Point owned by the AWS Account ID via `s3:DataAccessPointAccount`.

![S3 Bucket Policy Console View](https://github.com/GiovaniSerra/aws-labs/blob/main/s3-access-points-lab/images/s3%20-%20policy.png)

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "*"
            },
            "Action": "*",
            "Resource": [
                "arn:aws:s3:::<BUCKET_NAME>",
                "arn:aws:s3:::<BUCKET_NAME>/*"
            ],
            "Condition": {
                "StringEquals": {
                    "s3:DataAccessPointAccount": "<ACCOUNT_ID>"
                }
            }
        }
    ]
}
```
### Task 5: Testing & Validation via AWS CLI
Connected to the lab EC2 instance via SSM Session Manager and validated environment variables and policy behavior.

1. Environment Variable Setup & Local Verification
Setting environment variables ($bucket, $DentalAp, $XrayAp) and executing aws s3api list-objects-v2 with default instance credentials to inspect raw bucket contents.

![Local File Listing](https://github.com/GiovaniSerra/aws-labs/blob/main/s3-access-points-lab/images/ls.png)

![Terminal Session and Environment Variables](https://github.com/GiovaniSerra/aws-labs/blob/main/s3-access-points-lab/images/cd%20.png)

> Setting environment variables (`$bucket`, `$DentalAp`, `$XrayAp`) and executing `aws s3api list-objects-v2` with default instance credentials to inspect raw bucket contents.

2. Access Denial on Default Role Permissions
Without attached IAM identity permissions, requests from the instance role are blocked.

Calling s3:ListBucket from the EC2 instance role (InstanceIamRole) returns AccessDenied because no attached identity-based policy allows access.

![Access Denied for Instance Role](https://github.com/GiovaniSerra/aws-labs/blob/main/s3-access-points-lab/images/s3%20list%20-%20access%20denied.png)

#### 3. Direct Bucket Access vs. Access Point Access
Direct bucket access under dental_user profile fails as expected due to bucket delegation.

Executing aws s3api list-objects-v2 --bucket $bucket --profile dental_user returns AccessDenied, confirming direct access is blocked.

Listing objects via dental-ap succeeds:

Executing aws s3api list-objects-v2 --bucket $DentalAp --profile dental_user successfully returns the object key inventory.

![Direct Bucket Access Denied](https://github.com/GiovaniSerra/aws-labs/blob/main/s3-access-points-lab/images/s3%20list%20obj%20-%20prof%20dental_user%20-%20acc%20den.png)

Listing objects via `dental-ap` succeeds:

![List Objects via Dental Access Point](https://github.com/GiovaniSerra/aws-labs/blob/main/s3-access-points-lab/images/ls%20obj.png)

> Executing `aws s3api list-objects-v2 --bucket $DentalAp --profile dental_user` successfully returns the object key inventory.

4. Cross-Endpoint Security Enforcement
Attempting to list objects across an unauthorized Access Point is blocked.

Executing aws s3api list-objects-v2 --bucket $XrayAp --profile dental_user returns AccessDenied, confirming endpoint isolation.

![Dental User Blocked on Xray Access Point](https://github.com/GiovaniSerra/aws-labs/blob/main/s3-access-points-lab/images/ls%20obj%20-%20dental_user%20-%20not%20auth.png)

#### 5. ABAC Tag Enforcement & File Retrieval (s3:GetObject)
Downloading an authorized dental record via dental-ap:

Executing aws s3api get-object --bucket $DentalAp --key records_AVTR-...pdf using --profile dental_user. The request succeeds with HTTP 200, returning object metadata (ContentLength, AES256 encryption).

![GetObject Response](https://github.com/GiovaniSerra/aws-labs/blob/main/s3-access-points-lab/images/get%20obj%20-%20acc%20denied%20-%20not%20auth%20-%20profile%20dental_user.png)

---

## Cost Analysis & FinOps Considerations

Designing cloud architecture requires balancing security, operational efficiency, and financial optimization. Below is a cost breakdown of the services used in this lab setup based on standard AWS pricing:

### Service Cost Breakdown

| AWS Component | Pricing Model | Estimated Cost | Notes & FinOps Insights |
| :--- | :--- | :--- | :--- |
| **Amazon S3 Access Points** | **Free** | **$0.00** | There is **no additional charge** for creating or using S3 Access Points. You only pay for standard S3 requests and storage. |
| **Gateway VPC Endpoint** | **Free** | **$0.00** | Gateway Endpoints for S3 incur **no hourly fees and no data transfer charges**, making them ideal for high-throughput in-region S3 traffic. |
| **Amazon S3 Standard Storage** | Per GB / Month | ~$0.023 / GB | Standard S3 storage rates apply for medical PDF records and image files. |
| **S3 API Requests** | Per 1,000 requests | Negligible (`PUT`, `GET`, `LIST`) | Standard S3 API pricing applies (`s3:GetObject`, `s3:ListBucket`). Access Points do not add per-request overhead. |

---

## Infrastructure as Code with AWS CloudFormation

To improve repeatability, consistency, and alignment with Infrastructure as Code (IaC) best practices, the core infrastructure components of this lab—such as S3 Access Points, Access Point Policies, and Gateway VPC Endpoints—can be provisioned using AWS CloudFormation.

### Example CloudFormation Resources

#### 1. Provisioning an S3 Access Point and Access Point Policy
The following resources provision an S3 Access Point restricted to a specific VPC together with a dedicated Access Point policy that enforces Attribute-Based Access Control (ABAC) using Amazon S3 object tags:

```yaml
Resources:
  DentalAccessPoint:
    Type: AWS::S3::AccessPoint
    Properties:
      Name: dental-ap
      Bucket: !Ref MedicalRecordsBucket
      VpcConfiguration:
        VpcId: !Ref LabVpcId
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        IgnorePublicAcls: true
        BlockPublicPolicy: true
        RestrictPublicBuckets: true

  DentalAccessPointPolicy:
    Type: AWS::S3::AccessPointPolicy
    Properties:
      AccessPointName: !Ref DentalAccessPoint
      Policy:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              AWS: !Sub "arn:aws:iam::${AWS::AccountId}:user/DentalUser"
            Action: "s3:ListBucket"
            Resource: !Sub "arn:aws:s3:${AWS::Region}:${AWS::AccountId}:accesspoint/dental-ap"
          - Effect: Allow
            Principal:
              AWS: !Sub "arn:aws:iam::${AWS::AccountId}:user/DentalUser"
            Action: "s3:GetObject"
            Resource: !Sub "arn:aws:s3:${AWS::Region}:${AWS::AccountId}:accesspoint/dental-ap/object/*"
            Condition:
              StringEquals:
                "s3:ExistingObjectTag/recordtype": "dental"
```

Provisioning the S3 Gateway VPC Endpoint
Resource definition for the private Gateway Endpoint attached to the VPC private route table:

```yaml
S3GatewayEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      VpcId: !Ref LabVpcId
      ServiceName: !Sub "com.amazonaws.${AWS::Region}.s3"
      VpcEndpointType: Gateway
      RouteTableIds:
        - !Ref PrivateRouteTable
```
Note

These snippets illustrate the core CloudFormation resources used in this lab. A complete deployment would typically include the S3 bucket, bucket policy, IAM identities, and supporting networking resources.

### Benefits of Infrastructure as Code
Repeatable Deployments: Recreate the same infrastructure consistently across development, staging, and production environments.

Configuration Drift Prevention: Infrastructure definitions reduce the risk of manual configuration changes.

Version Control: Infrastructure changes can be reviewed, audited, and tracked through pull requests.

Scalability: Standardized templates simplify deployments across multiple AWS accounts and Regions.

---

## Key Financial Takeaways

* **Zero-Cost Access Control Boundary**: By shifting access governance from monolithic bucket policies to S3 Access Points, you achieve granular ABAC security without increasing AWS infrastructure costs.
* **Significant Savings via Gateway VPC Endpoint**: Choosing a Gateway VPC Endpoint over an Interface VPC Endpoint (AWS PrivateLink) saved **$0.01 per GB** in data processing fees plus **~$7.20/month per Availability Zone** in endpoint hourly charges.

---

## Lessons Learned & Best Practices
Decoupled Access Governance: S3 Access Points eliminate the complexity of maintaining massive, monolithic S3 bucket policies by shifting permission governance to endpoint-specific policies.

Perimeter Security: Requiring access through a Gateway VPC Endpoint ensures data access remains strictly within private cloud network boundaries without incurring additional network costs.

Dynamic ABAC Control: Leveraging s3:ExistingObjectTag enables fine-grained identity boundaries based on object attributes without changing baseline permissions.

Real-World Operational Impact
In enterprise environments with dozens of microservices or business analytics tools querying a central data lake, S3 Access Points significantly reduce operational friction. Instead of modifying a critical, shared bucket policy for every new onboarding request—risking service disruption—security teams can issue dedicated Access Points with scoped permissions, enforcing the principle of least privilege at scale.
