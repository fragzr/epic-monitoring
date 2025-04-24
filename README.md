# epic-monitoring
# AWS Multi-Account Monitoring Infrastructure

## Table of Contents
- [Architecture Overview](#architecture-overview)
- [Monitored Resources](#monitored-resources)
- [File Structure](#file-structure)
- [Deployment Options](#deployment-options)
- [Template Parameters](#template-parameters)
- [Resource Tags](#resource-tags)
- [Notes and Considerations](#notes-and-considerations)
- [Best Practices](#best-practices)
- [Limitations](#limitations)

## Architecture Overview

This solution provides automated monitoring and alerting for AWS resources across multiple accounts using CloudFormation StackSets. It consists of three main components:

### 1. Foundation Layer (SNS)
- SNS Topic for centralized alerting
- Email subscription for notifications
- Required SNS topic policies for CloudWatch and EventBridge

### 2. IAM Layer
- Lambda execution role with necessary permissions
- Managed policies for monitoring various AWS services
- Least privilege access model

### 3. Core Monitoring Layer
- Lambda function for resource discovery and alarm creation
- EventBridge rule for scheduled execution (5-minute intervals)
- Tag-based resource discovery
- Automated CloudWatch alarm management

## Monitored Resources

### EC2 Resources
- EC2 Instances
  * CPU Utilization
  * Memory Utilization (via CloudWatch Agent)
  * Status Checks
  * Credit Balance (T-series instances)
- EBS Volumes
  * Space Utilization
  * IOPS
  * Queue Length
  * Burst Balance (gp2/gp3)

### Network Resources
- Application Load Balancers
  * Active Connections
  * 5XX Errors
  * Response Time
- Network Load Balancers
  * Active Connections
  * TCP Reset Count
- VPN Connections
  * Tunnel Status
  * Bytes In/Out
- Direct Connect
  * Connection Status
  * Bandwidth Utilization
- Transit Gateways
  * Bytes Dropped
  * Packet Drop Count

### Security Resources
- WAF ACLs
  * Blocked Requests
  * Allowed Requests Rate
- Network Firewalls
  * Drop Packets
  * Alert Count

### Database & Storage
- RDS Instances
  * CPU Utilization
  * Freeable Memory
  * Read/Write Latency
- FSx Windows File Systems
  * Storage Capacity Utilization
  * Free Storage Capacity
- FSx NetApp ONTAP
  * Storage Capacity Utilization
  * Free Storage Capacity
  * Network Throughput

## File Structure

```
monitoring-infrastructure/
├── main-stack.yaml          # Main template for StackSet deployment
├── foundation-member.yaml    # SNS Topic and subscriptions
├── monitoring-iam.yaml      # IAM roles and policies
└── monitoring-core.yaml     # Lambda function and monitoring logic
```

## Template Parameters

### main-stack.yaml
Parameter | Description | Required
----------|-------------|----------
ProjectName | Project name for resource tagging | Yes
NotificationEmail | Email address for notifications | Yes
S3BucketName | S3 bucket containing nested templates | Yes
S3KeyPrefix | S3 key prefix for nested templates | No

### Account Environment Mapping
Account ID | Environment
-----------|------------
841125194518 | PROD
461605441988 | NONPROD
852325766660 | SHARED
253647676190 | TRAIN
227932815323 | READONLY

## Resource Tags

Resources are discovered based on tags defined in SSM Parameters:
- /epic/monitoring/ec2-tags
- /epic/monitoring/ebs-tags
- /epic/monitoring/alb-tags
- /epic/monitoring/nlb-tags
- /epic/monitoring/vpn-tags
- /epic/monitoring/dx-tags
- /epic/monitoring/tgw-tags
- /epic/monitoring/waf-tags
- /epic/monitoring/nfw-tags
- /epic/monitoring/rds-tags
- /epic/monitoring/fsxwin-tags
- /epic/monitoring/fsxnetapp-tags

## Notes and Considerations

### Thresholds by Environment
Resource | PROD | NONPROD | SHARED | TRAIN | READONLY
---------|------|---------|---------|--------|----------
CPU Utilization | 80% | 85% | 85% | 90% | 90%
Memory Utilization | 80% | 85% | 85% | 90% | 90%
Storage Utilization | 75% | 80% | 80% | 85% | 85%

### Lambda Function
- Execution Frequency: Every 5 minutes
- Timeout: 900 seconds
- Memory: 512 MB
- Python Runtime: 3.9

## Best Practices
1. Tag resources consistently across accounts
2. Review and confirm SNS subscriptions
3. Monitor Lambda execution metrics
4. Regularly review CloudWatch Log groups
5. Maintain SSM Parameter tags configuration

## Limitations
1. Maximum resources per Lambda execution
2. CloudWatch API rate limits
3. CloudWatch Alarms per account limits
4. SNS topic subscription limits
5. Resource tagging requirements
