
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
- CloudWatch Dashboard creation

## Monitored Resources

### EC2 Resources
- EC2 Instances
  * CPU Utilization
  * Memory Utilization (via CloudWatch Agent)
  * Status Checks
  * Credit Balance (T-series instances)
  * EBS Status Check
- EBS Volumes
  * Space Utilization
  * IOPS
  * Queue Length
  * Burst Balance (gp2/gp3)
  * Volume Detachment Events

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
├── monitoring-core.yaml     # Lambda function and monitoring logic
└── monitoring-iam.yaml      # IAM roles and policies
```

## Template Parameters

### main-stack.yaml
Parameter | Description | Required
----------|-------------|----------
ProjectName | Project name for resource tagging | Yes
NotificationEmail | Email address for notifications | Yes
S3BucketName | S3 bucket containing nested templates | Yes
S3KeyPrefix | S3 key prefix for nested templates | No

### Account Environment Mapping - Please update accounts number accordingly
Account ID | Environment
-----------|------------
<Account ID> | PROD
<Account ID> | NONPROD
<Account ID> | SHARED
<Account ID> | TRAIN
<Account ID> | READONLY

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

#### Compute Resources
Resource Metric | PROD | NONPROD | SHARED | TRAIN | READONLY
----------------|------|---------|---------|--------|----------
EC2 CPU Utilization | 80% | 85% | 85% | 90% | 90%
EC2 Memory Utilization | 80% | 85% | 85% | 90% | 90%
EC2 CPU Credit Balance | <20 | <20 | <20 | <20 | <20
EC2 Status Check | ≥1 failures | ≥1 failures | ≥1 failures | ≥1 failures | ≥1 failures
EBS Status Check | >0 failures | >0 failures | >0 failures | >0 failures | >0 failures

#### Storage Resources
Resource Metric | PROD | NONPROD | SHARED | TRAIN | READONLY
----------------|------|---------|---------|--------|----------
EBS Space Utilization | <20% free | <15% free | <15% free | <10% free | <10% free
EBS IOPS (io2) | >10000 | >10000 | >10000 | >10000 | >10000
EBS IOPS (other types) | >3000 | >3000 | >3000 | >3000 | >3000
EBS Queue Length | >1 | >1 | >1 | >1 | >1
FSx Storage Capacity | 80% | 85% | 85% | 90% | 90%
FSx Free Storage | <10GB | <10GB | <10GB | <10GB | <10GB
FSx ONTAP Network Throughput | 80% | 85% | 85% | 90% | 90%

#### Load Balancer Metrics
Resource Metric | PROD | NONPROD | SHARED | TRAIN | READONLY
----------------|------|---------|---------|--------|----------
ALB Active Connections | >5000 | >2000 | >2000 | >1000 | >1000
ALB 5XX Errors | >10 | >20 | >20 | >50 | >50
ALB Response Time | >1s | >2s | >2s | >5s | >5s
NLB Active Connections | >5000 | >2000 | >2000 | >1000 | >1000
NLB TCP Reset Count | >100 | >200 | >200 | >500 | >500

#### Network Resources
Resource Metric | PROD | NONPROD | SHARED | TRAIN | READONLY
----------------|------|---------|---------|--------|----------
VPN Tunnel Status | <0 | <0 | <0 | <0 | <0
VPN Bytes In | >100MB | >100MB | >100MB | >100MB | >100MB
Direct Connect Status | <0 | <0 | <0 | <0 | <0
Direct Connect Bandwidth | 80% | 85% | 85% | 90% | 90%
Transit Gateway Bytes Dropped | >1000 | >1000 | >1000 | >1000 | >1000
Transit Gateway Packet Drops | >1000 | >1000 | >1000 | >1000 | >1000

#### Security Resources
Resource Metric | PROD | NONPROD | SHARED | TRAIN | READONLY
----------------|------|---------|---------|--------|----------
WAF Blocked Requests | >1000 | >2000 | >2000 | >5000 | >5000
WAF Allowed Requests Rate | >10000 | >20000 | >20000 | >50000 | >50000
Network Firewall Dropped Packets | >1000 | >2000 | >2000 | >5000 | >5000
Network Firewall Alert Count | >100 | >200 | >200 | >500 | >500

#### Database Resources
Resource Metric | PROD | NONPROD | SHARED | TRAIN | READONLY
----------------|------|---------|---------|--------|----------
RDS CPU Utilization | 80% | 85% | 85% | 90% | 90%
RDS Freeable Memory | ≤2048MB | ≤2048MB | ≤2048MB | ≤2048MB | ≤2048MB
RDS Read Latency | >0.02s | >0.02s | >0.02s | >0.02s | >0.02s
RDS Write Latency | >0.02s | >0.02s | >0.02s | >0.02s | >0.02s

### Evaluation Periods and Duration
Metric Type | Evaluation Periods | Period Duration
------------|-------------------|----------------
EC2 Metrics | 2 periods | 300 seconds
EBS Metrics | 2 periods | 300 seconds
Load Balancer Metrics | 2 periods | 300 seconds
Network Metrics | 1-2 periods | 300 seconds
Security Metrics | 2 periods | 300 seconds
Database Metrics | 3-5 periods | 300 seconds
Storage Metrics | 2 periods | 900 seconds

### Missing Data Treatment
Metric Type | Missing Data Treatment
------------|---------------------
EC2 CPU/Memory | missing
EC2 Status Checks | breaching
EBS Metrics | missing/ignore
Load Balancer Metrics | ignore
Network Status | breaching
Network Metrics | ignore
Security Metrics | ignore
Database Metrics | missing
Storage Metrics | missing


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

## File Functionalities

### main-stack.yaml
- Defines the main CloudFormation stack
- Sets up SSM Parameters for tag configurations
- Creates nested stacks for IAM and core monitoring components
- Configures account environment mappings

### monitoring-core.yaml
- Defines the core monitoring Lambda function
- Sets up SNS topic for notifications
- Creates EventBridge rule for scheduled Lambda execution
- Implements resource discovery and alarm creation logic
- Handles CloudWatch dashboard creation

### monitoring-iam.yaml
- Defines IAM roles and policies for the monitoring Lambda function
- Implements least privilege access model
- Provides necessary permissions for resource discovery, alarm creation, and notifications
