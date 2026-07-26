
# AWS Logging & Security Monitoring: CloudTrail & CloudWatch Alerts

## Overview
This project demonstrates the implementation of a continuous security monitoring and observability solution on AWS. The primary objective is to capture real-time audit events, detect failed authentication attempts on the AWS Management Console, and trigger automated email notifications before potential security incidents escalate.

The solution also covers incident investigation and digital forensics using structured queries in Amazon CloudWatch Logs Insights.

## Architecture Diagram

![Diagrama](https://github.com/GiovaniSerra/aws-labs/blob/main/security-observability/cloudtrail-cloudwatch-alerts/ar.jpg)

Workflow
User Authentication Attempt: A user attempts to log in to the AWS Management Console with invalid credentials.

Event Capture (AWS CloudTrail): CloudTrail captures the failed login event (ConsoleLogin) across the AWS account.

Log Delivery (Amazon CloudWatch Logs): Events are streamed in real time to a dedicated CloudWatch Log Group (CloudTrailLogGroup).

Pattern Detection (Metric Filter): A metric filter scans log streams for failed authentication events and increments the ConsoleLoginFailureCount metric.

Threshold Evaluation (CloudWatch Alarm): An alarm triggers when the failure count reaches or exceeds 3 attempts within a 5-minute window.

Automated Alerting (Amazon SNS): The alarm transitions to the ALARM state and publishes a message to an SNS topic, delivering an immediate email notification to the security team.

Forensic Analysis (CloudWatch Logs Insights): Administrators perform interactive queries over log data to analyze incident source details (IP address, user ARN, region).

## AWS Services Used

AWS CloudTrail: Captures account activity and API calls, streaming log events to CloudWatch Logs for centralized auditing.

Amazon CloudWatch Logs: Serves as the central repository for log storage, retention, and real-time event filtering.

Amazon CloudWatch Metrics & Alarms: Extracts quantitative metrics from log streams and monitors thresholds to trigger automated responses.

Amazon Simple Notification Service (SNS): Handles pub/sub messaging to deliver real-time security alerts via email.

Amazon CloudWatch Logs Insights: Provides an interactive query engine to perform rapid forensic analysis and incident investigation.

## Step-by-Step Implementation

Task 1: Create a CloudTrail Trail with CloudWatch Logs Enabled
Configured an AWS CloudTrail trail (MyLabCloudTrail) to log management events across the AWS account.

Integrated the trail with Amazon CloudWatch Logs by targeting a dedicated Log Group (CloudTrailLogGroup).

Associated the IAM role LabCloudTrailRole to grant CloudTrail the necessary permissions to write log streams.

Task 2: Create and Subscribe to an Amazon SNS Topic
Created a Standard Amazon SNS topic named MySNSTopic to manage alert distribution.

Configured access policies for message publishers and subscribers.

Added an Email subscription to the topic and confirmed the registration through the confirmation email.

Task 3: Create a CloudWatch Alarm Based on a Metric Filter
Applied a custom Metric Filter to the CloudWatch Log Group to identify failed login attempts.

Filter Pattern:

```
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

Analysis Results:  

Aggregated Insights: The query isolated failed console authentication events and grouped them by source IP address, failure reason, AWS region, and targeted IAM user ARN.

Incident Attribution: Confirmed the exact IP address and user identity associated with the security event, enabling targeted incident response and rule validation.
## Key Takeaways
Reduced MTTD (Mean Time to Detect): Automated detection of critical security events without relying on manual log reviews.

Traceability & Governance: Centralized audit logs aligned with AWS security best practices.

Rapid Incident Response: Ability to quickly isolate suspicious IP addresses and affected user accounts using structured query capabilities.
