
# AWS Logging & Security Monitoring: CloudTrail & CloudWatch Alerts

## Overview
This project demonstrates the implementation of a continuous security monitoring and observability solution on AWS. The primary objective is to capture real-time audit events, detect failed authentication attempts on the AWS Management Console, and trigger automated email notifications before potential security incidents escalate.

The solution also covers incident investigation and digital forensics using structured queries in Amazon CloudWatch Logs Insights.

> **AWS Architecture Note — Management vs. Data Events:**
> CloudTrail categorizes events into **Management Events** (control plane operations like IAM changes, EC2 creation, `ConsoleLogin`) and **Data Events** (data plane operations like S3 `GetObject` or Lambda `Invoke`).
> This project focuses on **Management Events**, which provide comprehensive administrative auditing without incurring the higher ingestion costs associated with high-volume Data Events.

---

## Architecture Diagram

![Architecture Diagram](https://github.com/GiovaniSerra/aws-labs/blob/main/security-observability/cloudtrail-cloudwatch-alerts/ar.jpg?raw=true)

### Workflow
1. **User Authentication Attempt:** A user attempts to log in to the AWS Management Console with invalid credentials.
2. **Event Capture (AWS CloudTrail):** CloudTrail captures the failed login event (`ConsoleLogin`) across the AWS account.
3. **Log Delivery (Amazon CloudWatch Logs):** Events are streamed in real time to a dedicated CloudWatch Log Group (`CloudTrailLogGroup`).
4. **Pattern Detection (Metric Filter):** A metric filter scans log streams for failed authentication events and increments the `ConsoleLoginFailureCount` metric.
5. **Threshold Evaluation (CloudWatch Alarm):** An alarm triggers when the failure count reaches or exceeds 3 attempts within a 5-minute window.
6. **Automated Alerting (Amazon SNS):** The alarm transitions to the `ALARM` state and publishes a message to an SNS topic, delivering an immediate email notification to the security team.
7. **Forensic Analysis (CloudWatch Logs Insights):** Administrators perform interactive queries over log data to analyze incident source details (IP address, user ARN, region).

### Expected Pipeline Latency
In cloud security operations, understanding delivery SLAs prevents false assumptions about system behavior:
* **CloudTrail to CloudWatch Logs:** 30 seconds to 5 minutes (depending on AWS event batching).
* **Metric Filter to CloudWatch Alarm:** Evaluated continuously based on the 5-minute aggregation window.
* **CloudWatch Alarm to SNS Email:** Delivered within seconds after the metric crosses the defined threshold.

---

## AWS Services Used

* **AWS CloudTrail:** Captures account activity and API calls, streaming log events to CloudWatch Logs for centralized auditing.
* **Amazon CloudWatch Logs:** Serves as the central repository for log storage, retention, and real-time event filtering.
* **Amazon CloudWatch Metrics & Alarms:** Extracts quantitative metrics from log streams and monitors thresholds to trigger automated responses.
* **Amazon Simple Notification Service (SNS):** Handles pub/sub messaging to deliver real-time security alerts via email.
* **Amazon CloudWatch Logs Insights:** Provides an interactive query engine to perform rapid forensic analysis and incident investigation.

---

## Infrastructure as Code (IaC) Deployment

This entire security monitoring architecture can be automatically provisioned using the included **AWS CloudFormation** template [template.yaml](https://github.com/GiovaniSerra/aws-labs/blob/main/security-observability/cloudtrail-cloudwatch-alerts/template.yaml).

### Deployment via AWS CLI

Run the following command to deploy the stack, replacing the email parameter with your target notification address:

```bash
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name cloudtrail-security-monitoring \
  --parameter-overrides NotificationEmail="your-email@example.com" \
  --capabilities CAPABILITY_NAMED_IAM
```
---

## Step-by-Step Implementation

### Task 1: Create a CloudTrail Trail with CloudWatch Logs Enabled
* Configured an AWS CloudTrail trail (`MyLabCloudTrail`) to log management events across the AWS account.
* Integrated the trail with Amazon CloudWatch Logs by targeting a dedicated Log Group (`CloudTrailLogGroup`).
* Associated the IAM role `LabCloudTrailRole` to grant CloudTrail the necessary permissions to write log streams.

### Task 2: Create and Subscribe to an Amazon SNS Topic
* Created a Standard Amazon SNS topic named `MySNSTopic` to manage alert distribution.
* Configured access policies for message publishers and subscribers.
* Added an Email subscription to the topic and confirmed the registration through the confirmation email.

### Task 3: Create a CloudWatch Alarm Based on a Metric Filter
Applied a custom Metric Filter to the CloudWatch Log Group to identify failed login attempts.

**Filter Pattern (JSON):**
```json
{ ($.eventName = ConsoleLogin) && ($.errorMessage = "Failed authentication") }
```

Metric Configuration:

Filter Name: ConsoleLoginErrors

Metric Namespace: CloudTrailMetrics

Metric Name: ConsoleLoginFailureCount

Metric Value: 1

Alarm Configuration (FailedLogins):

Threshold Condition: ConsoleLoginFailureCount >= 3 for 1 datapoint within 5 minutes.

Notification Action: Send message to MySNSTopic upon transitioning to the ALARM state.

### Task 4: Validation & Incident Simulation

* **Testing:** Simulated unauthorized access by attempting multiple console logins with incorrect credentials using a test IAM user.
* **Data Processing:** CloudTrail captured the failed login events, streamed them to CloudWatch Logs, and updated the `ConsoleLoginFailureCount` metric.

![CloudWatch Metric](https://github.com/GiovaniSerra/aws-labs/blob/main/security-observability/cloudtrail-cloudwatch-alerts/metrics.png?raw=true)

* **Alarm Triggering:** The alarm state shifted from `OK` / `INSUFFICIENT_DATA` to `ALARM` after crossing the defined threshold (`>= 3` failed logins in 5 minutes).

![Alarms Overview](https://github.com/GiovaniSerra/aws-labs/blob/main/security-observability/cloudtrail-cloudwatch-alerts/alarms.png?raw=true)

* **Alarm Evaluation & Execution Verification:** The detailed view confirms the condition logic, while the history log verifies that the automated action to trigger SNS executed successfully.

| Alarm Details | Execution History |
| :---: | :---: |
| ![Alarm Details](https://github.com/GiovaniSerra/aws-labs/blob/main/security-observability/cloudtrail-cloudwatch-alerts/FailedLogins%20-%20details.png?raw=true) | ![Alarm History](https://github.com/GiovaniSerra/aws-labs/blob/main/security-observability/cloudtrail-cloudwatch-alerts/FailedLogins%20-%20history.png?raw=true) |

* **Alert Delivery:** Verified the receipt of an automated email notification containing alarm context, AWS Account ID, Region, and timestamp details.

![SNS Email Alert](https://github.com/GiovaniSerra/aws-labs/blob/main/security-observability/cloudtrail-cloudwatch-alerts/sns%20-%20email.png?raw=true)

## Forensic Analysis with CloudWatch Logs Insights
After receiving the alarm notification, an interactive investigation was conducted in CloudWatch Logs Insights to analyze the underlying log events and identify the source of the failed attempts.

Query Executed (SQL):

```
filter eventSource="signin.amazonaws.com" and eventName="ConsoleLogin" and responseElements.ConsoleLogin="Failure"
| stats count(*) as Total_Count by sourceIPAddress as Source_IP, errorMessage as Reason, awsRegion as AWS_Region, userIdentity.arn as IAM_Arn
```

## Cost & Resource Estimation
This architecture is optimized for cost efficiency and fits entirely within the AWS Free Tier for evaluation purposes:

AWS CloudTrail: First copy of Management Events is Free.

Amazon CloudWatch Logs: Free Tier includes 5 GB of log ingestion and 5 GB of log storage.

Amazon CloudWatch Alarms: Free Tier includes 10 custom metrics and 10 alarm metrics.

Amazon SNS: Free Tier includes 1,000,000 requests and 1,000 email notifications per month.

Estimated running cost in a sandbox environment: $0.00 / month.


## Production vs. Laboratory Considerations
While this lab establishes the core auditing pipeline, implementing this pattern in an enterprise production environment requires additional hardening:

Multi-Region & Multi-Account: Enable CloudTrail across all AWS regions and centralize logs into a dedicated Security/Log Archive AWS Account using AWS Organizations.

Log Integrity & Encryption: Enable Log File Validation in CloudTrail to detect tampering and enforce S3 SSE-KMS encryption.

Retention Policies: Set explicit CloudWatch Logs retention policies (e.g., 365+ days for compliance) or transition older logs to S3 Glacier.

Expanded Security Alarms: Deploy alarms for critical events such as Root account usage, MFA disabling, IAM policy modifications, and unauthorized API calls.

Modern Incident Response Integration: Route CloudWatch Alarms or EventBridge rules to AWS Security Hub, PagerDuty, or Slack webhooks via AWS Lambda for immediate SOC notification.

Analysis Results:  

Aggregated Insights: The query isolated failed console authentication events and grouped them by source IP address, failure reason, AWS region, and targeted IAM user ARN.

Incident Attribution: Confirmed the exact IP address and user identity associated with the security event, enabling targeted incident response and rule validation.

## Key Takeaways
Reduced MTTD (Mean Time to Detect): Automated detection of critical security events without relying on manual log reviews.

Traceability & Governance: Centralized audit logs aligned with AWS security best practices.

Rapid Incident Response: Ability to quickly isolate suspicious IP addresses and affected user accounts using structured query capabilities.
