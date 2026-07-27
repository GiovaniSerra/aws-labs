# AWS Hands-On Lab: Managing Access at Scale with Amazon S3 Access Points

## Overview
This laboratory project demonstrates how to manage data access policies at scale using **Amazon S3 Access Points**. 

A medical imaging company stores sensitive patient records (dental files, prescriptions, and X-ray images) in Amazon S3. To adhere to the principle of least privilege, specific IAM identities need restricted access based on data category and network origin.

Using S3 Access Points, bucket policies, and VPC endpoints, access control was delegated to dedicated endpoints with strict tag-based conditions and network perimeter controls.

---

## Architecture Diagram



## Key Objectives
Explain the role of Amazon S3 Access Points in scaling access management.

Implement custom Access Point policies with Attribute-Based Access Control (ABAC) using s3:ExistingObjectTag.

Restrict S3 traffic through a Gateway VPC Endpoint for private network isolation.

Configure an S3 Bucket Policy to delegate access control directly to S3 Access Points.

## Step-by-Step Implementation
Task 1: Environment Assessment
IAM Identities: Inspected DentalUser and XrayUser. Neither identity had direct attached policies or group memberships.

S3 Data Classification: Inspected the pre-populated S3 bucket (medical-records-*). Objects are classified using tags:

Dental files: recordtype = dental

X-ray files: recordtype = xray

Bucket Protection: Confirmed that Block Public Access is fully active across the bucket.

Task 2: Provision S3 Access Points
1. Dental Access Point (dental-ap)
Configured an S3 Access Point attached to the VPC with an inline policy granting DentalUser permission to list and retrieve objects tagged with recordtype: dental.

Name: dental-ap

Network Origin: Virtual Private Cloud (VPC)

VPC ID: <LAB_VPC_ID>

Block Public Access: Enabled

```JSON
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
2. X-ray Access Point (xray-ap)
Configured a dedicated Access Point allowing XrayUser to access only objects tagged with recordtype: xray.

Name: xray-ap

Network Origin: Virtual Private Cloud (VPC)

VPC ID: <LAB_VPC_ID>

Block Public Access: Enabled

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
Task 3: Create Gateway VPC Endpoint
Provisioned a Gateway VPC Endpoint to route traffic from the EC2 instance inside the VPC to the S3 bucket and its access points privately using AWS PrivateLink architecture.

Service: com.amazonaws.<REGION>.s3 (Gateway)

VPC: Lab VPC

Route Table: Lab Subnet (Public)

```JSON
{
    "Version": "2008-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:ListBucket",
            "Resource": [
                "arn:aws:s3:::<BUCKET_NAME>",
                "arn:aws:s3:<REGION>:<ACCOUNT_ID>:accesspoint/dental-ap",
                "arn:aws:s3:<REGION>:<ACCOUNT_ID>:accesspoint/xray-ap"
            ]
        },
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": [
                "arn:aws:s3:::<BUCKET_NAME>/*",
                "arn:aws:s3:<REGION>:<ACCOUNT_ID>:accesspoint/dental-ap/object/*",
                "arn:aws:s3:<REGION>:<ACCOUNT_ID>:accesspoint/xray-ap/object/*"
            ]
        }
    ]
}
```

Task 4: Delegate Bucket Access Control
Applied a bucket policy to medical-records-* delegating control directly to any Access Point owned by the AWS Account ID (s3:DataAccessPointAccount).

```JSON
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
Task 5: Testing & Validation via AWS CLI
Connected to the lab EC2 instance using AWS Systems Manager (SSM) Session Manager.

1. Environmental Variables Setup

```Bash
bucket="<BUCKET_NAME>"
DentalAp="arn:aws:s3:<REGION>:<ACCOUNT_ID>:accesspoint/dental-ap"
XrayAp="arn:aws:s3:<REGION>:<ACCOUNT_ID>:accesspoint/xray-ap"
```

2. Direct Bucket Access vs. Access Point Access
Direct bucket requests under dental_user profile fail as expected due to bucket delegation:

```Bash
aws s3api list-objects-v2 --bucket $bucket --profile dental_user
```

# Output: An error occurred (AccessDenied)
Listing objects via dental-ap succeeds:

```Bash
aws s3api list-objects-v2 --bucket $DentalAp --profile dental_user
# Output: Successfully lists objects
```

3. ABAC Tag Enforcement
Downloading a dental record via dental-ap succeeds:

```Bash
aws s3api get-object --bucket $DentalAp --key records_AVTR-7531421564.pdf records_AVTR-7531421564.pdf --profile dental_user
```

# Output: HTTP 200 OK
Attempting to download an X-ray file via dental-ap fails:

```Bash
aws s3api get-object --bucket $DentalAp --key xray_PRAM-5741336854.png xray_PRAM-5741336854.png --profile dental_user
# Output: An error occurred (AccessDenied)
```

4. Cross-Endpoint Security
XrayUser downloading an X-ray file via xray-ap succeeds:

```Bash
aws s3api get-object --bucket $XrayAp --key xray_WASD-8749317132.png xray_WASD-8749317132.png --profile xray_user
# Output: HTTP 200 OK
```

XrayUser querying dental-ap fails:

```Bash
aws s3api list-objects-v2 --bucket $DentalAp --profile xray_user
# Output: An error occurred (AccessDenied)
```

## Lessons Learned & Best Practices
Decoupled Access Governance: S3 Access Points eliminate the complexity of maintaining massive, monolithic S3 bucket policies by shifting permission governance to endpoint-specific policies.

Perimeter Security: Requiring access through a VPC Endpoint ensures data access remains strictly within private cloud network boundaries.

Dynamic ABAC Control: Leveraging s3:ExistingObjectTag enables fine-grained identity boundaries based on object attributes without changing baseline permissions.
