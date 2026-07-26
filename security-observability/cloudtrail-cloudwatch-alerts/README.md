
# AWS Logging & Security Monitoring: CloudTrail & CloudWatch Alerts

## Overview
This laboratory demonstrates the implementation of a continuous security monitoring and observability solution on AWS. The primary objective is to capture real-time audit events, monitor failed authentication attempts on the AWS Management Console, and trigger automated email notifications before potential security incidents escalate.

The solution also covers incident investigation and digital forensics using structured queries in Amazon CloudWatch Logs Insights.

## Solution Architecture

[ AWS Console Login ]
          │
          ▼ (Failed Attempt)
[ AWS CloudTrail ] ──► Records API audit logs
          │
          ▼
[ Amazon CloudWatch Logs ] ──► Log Group: CloudTrailLogGroup
          │
          ▼
[ Metric Filter ] ──► Filters event: ConsoleLogin with "Failed authentication"
          │
          ▼
[ CloudWatch Alarm ] ──► Rule: ≥ 3 failures in 5 minutes
          │
          ▼
[ Amazon SNS Topic ] ──► Email notification to the security team

## AWS Services Used
AWS CloudTrail: Continuous recording of account activity and API calls.

Amazon CloudWatch Logs: Centralized log storage, management, and retention.

Amazon CloudWatch Metrics & Alarms: Creation of custom metrics and automated alarm rules based on operational thresholds.

Amazon Simple Notification Service (SNS): Pub/Sub messaging service for sending automated email alerts.

CloudWatch Logs Insights: Interactive query engine for fast log analysis and incident response.

## Step-by-Step Implementation

1. Audit Trail Configuration
Reviewed management events captured by CloudTrail across the AWS account.

Configured the trail MyLabCloudTrail integrated with CloudWatch Logs targeting the CloudTrailLogGroup.

Associated the dedicated IAM role LabCloudTrailRole to grant appropriate log writing permissions.

2. Notification Infrastructure Setup (Amazon SNS)
Created a Standard SNS topic named MySNSTopic.

Configured access policies for publishers and subscribers.

Created an Email protocol subscription and confirmed subscription to receive infrastructure alerts.

3. Metric Filter & CloudWatch Alarm Creation
To detect potential brute-force attacks or unauthorized access attempts, a custom metric filter was applied to the CloudTrail logs.

Filter Pattern:

{ ($.eventName = ConsoleLogin) && ($.errorMessage = "Failed authentication") }

Metric Configuration:

Filter Name: ConsoleLoginErrors

Metric Namespace: CloudTrailMetrics

Metric Name: ConsoleLoginFailureCount

Metric Value: 1

Alarm Rule (FailedLogins):

Condition: Triggers whenever the sum of ConsoleLoginFailureCount is greater than or equal to 3 within a 5-minute evaluation period.

Action: Sends an automated notification to MySNSTopic.
Validation & Testing
Incident Simulation: Performed multiple failed login attempts using incorrect credentials for the test IAM user test.

Data Collection & Visualization: Failed events were logged by CloudTrail, streamed to CloudWatch Logs, and tracked on the ConsoleLoginFailureCount metric graph.

Alarm Triggering: The alarm state transitioned from OK to ALARM, invoking the SNS notification.

Alert Delivery: Verified the receipt of an alert email containing critical event details, including AWS Account ID, Region, Timestamp, and Threshold state.

## Forensic Analysis with CloudWatch Logs Insights
Following the alarm notification, an investigation was conducted on the log data to determine the origin and impact of the failed login attempts.

Query Executed:

filter eventSource="signin.amazonaws.com" and eventName="ConsoleLogin" and responseElements.ConsoleLogin="Failure"
| stats count(*) as Total_Count by sourceIPAddress as Source_IP, errorMessage as Reason, awsRegion as AWS_Region, userIdentity.arn as IAM_Arn

Result: The query successfully aggregated and displayed the number of failed attempts grouped by source IP address, failure reason, AWS region, and targeted IAM user ARN.

## Key Takeaways
Reduced MTTD (Mean Time to Detect): Automated detection of critical security events without relying on manual log reviews.

Traceability & Governance: Centralized audit logs aligned with AWS security best practices.

Rapid Incident Response: Ability to quickly isolate suspicious IP addresses and affected user accounts using structured query capabilities.
