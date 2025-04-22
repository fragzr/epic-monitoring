# epic-monitoring
# AWS Multi-Account Monitoring Infrastructure

## Table of Contents
- [Architecture Overview](#architecture-overview)
- [Monitored Resources](#monitored-resources)
- [File Structure](#file-structure)
- [Deployment Options](#deployment-options)
  - [Single Account Deployment](#single-account-deployment)
  - [Multi-Account Deployment via StackSet](#multi-account-deployment-via-stackset)
- [Important Parameters](#important-parameters)
- [Notes and Considerations](#notes-and-considerations)
- [Best Practices](#best-practices)
- [Limitations](#limitations)
- [Support](#support)

## Architecture Overview

This solution provides automated monitoring and alerting for AWS resources across multiple accounts. It consists of three main components:

### 1. Foundation Layer (SNS)
- SNS Topic for alerts
- Email subscriptions
- Required topic policies

### 2. IAM Layer
- Lambda execution role
- Required permissions for monitoring services

### 3. Core Monitoring Layer
- Lambda function for resource discovery and alarm creation
- EventBridge rule for scheduled execution
- CloudWatch Log groups
- CloudWatch Alarms

## Monitored Resources

### EC2 Instances
- CPU Utilization
- Status Checks
- Network Utilization
- CPU Credit Balance (t-series instances)

### EBS Volumes
- Queue Length
- IOPS
- Burst Balance (gp2/gp3)

### Network Resources
- VPN Connections
- VPN Tunnels

### File Structure
```
monitoring-infrastructure/
├── foundation-member.yaml    # SNS Topic and related resources
├── monitoring-iam.yaml       # IAM roles and policies
├── monitoring-core.yaml      # Lambda function and monitoring logic
└── main-stack.yaml          # Main stack for StackSet deployment
```


## Deployment Options

### Single Account Deployment

#### Prerequisites
* AWS CLI configured with appropriate permissions
* S3 bucket for template storage (optional)

#### Deployment Steps

1. Deploy IAM Stack
```bash
aws cloudformation create-stack \
  --stack-name epic-monitoring-iam \
  --template-body file://monitoring-iam.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameters \
    ParameterKey=Environment,ParameterValue=prod \
    ParameterKey=ProjectName,ParameterValue=Epic
```

2. Deploy Foundation Stack
```bash
aws cloudformation create-stack \
  --stack-name epic-monitoring-foundation \
  --template-body file://foundation-member.yaml \
  --parameters \
    ParameterKey=Environment,ParameterValue=prod \
    ParameterKey=ProjectName,ParameterValue=Epic \
    ParameterKey=NotificationEmail,ParameterValue=your-email@domain.com
```

3. Deploy Core Stack
```bash
aws cloudformation create-stack \
  --stack-name epic-monitoring-core \
  --template-body file://monitoring-core.yaml \
  --capabilities CAPABILITY_IAM \
  --parameters \
    ParameterKey=Environment,ParameterValue=prod \
    ParameterKey=ProjectName,ParameterValue=Epic \
    ParameterKey=MonitoringRoleArn,ParameterValue=<IAM-Stack-Role-ARN> \
    ParameterKey=ResourceTagKey,ParameterValue=Environment \
    ParameterKey=ResourceTagValue,ParameterValue=PROD
```

### Multi-Account Deployment via StackSet

#### Prerequisites
* Management account access
* S3 bucket for template storage
* AWS Organizations setup
* StackSet administration and execution roles

#### Setup Steps

1. Create Required Roles

In Management Account (stackset-admin-role.yaml):
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Resources:
  AdministrationRole:
    Type: 'AWS::IAM::Role'
    Properties:
      RoleName: AWSCloudFormationStackSetAdministrationRole
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: cloudformation.amazonaws.com
            Action: 'sts:AssumeRole'
      ManagedPolicyArns:
        - 'arn:aws:iam::aws:policy/AdministratorAccess'
```

In Member Accounts (stackset-execution-role.yaml):
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Parameters:
  AdministratorAccountId:
    Type: String
Resources:
  ExecutionRole:
    Type: 'AWS::IAM::Role'
    Properties:
      RoleName: AWSCloudFormationStackSetExecutionRole
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              AWS:
                - !Sub 'arn:aws:iam::${AdministratorAccountId}:role/AWSCloudFormationStackSetAdministrationRole'
            Action: 'sts:AssumeRole'
      ManagedPolicyArns:
        - 'arn:aws:iam::aws:policy/AdministratorAccess'
```

2. Prepare S3 Bucket
```bash
# Create bucket
aws s3 mb s3://epic-monitoring-cfn-template

# Upload templates
aws s3 cp foundation-member.yaml s3://epic-monitoring-cfn-template/
aws s3 cp monitoring-iam.yaml s3://epic-monitoring-cfn-template/
aws s3 cp monitoring-core.yaml s3://epic-monitoring-cfn-template/
aws s3 cp main-stack.yaml s3://epic-monitoring-cfn-template/
```

3. Deploy StackSet
   * Navigate to CloudFormation → StackSets
   * Click "Create StackSet"
   * Select "Self-service permissions"
   * Upload main-stack.yaml
   * Fill in parameters:
     * ProjectName: Epic
     * NotificationEmail: your-email@domain.com
     * ResourceTagKey: Environment
   * Add target accounts or organizational units
   * Select deployment regions

## Important Parameters

### Common Parameters
Parameter | Description | Default
-----------|-------------|----------
ProjectName | Resource naming prefix | Epic
Environment | Environment type | prod/nonprod/shared/train/readonly
NotificationEmail | Email for alerts | -

### Monitoring Parameters
Parameter | Description | Default
-----------|-------------|----------
CPUThreshold | EC2 CPU utilization threshold | 95%
MemoryThreshold | Memory utilization threshold | 85%
DiskThreshold | Disk utilization threshold | 85%
BurstBalanceThreshold | EBS burst balance threshold | 20%
NetworkThreshold | Network utilization threshold | 80%
IOPSThreshold | IOPS threshold count | 5000

### Resource Tags
Resources must be tagged with:
* Key: Environment (or custom key specified in ResourceTagKey)
* Value: PROD/NONPROD/SHARED/TRAIN/READONLY (based on environment)

## Notes and Considerations

### 1. Order of Deployment
* IAM roles must be created first
* Foundation layer (SNS) second
* Core monitoring last

### 2. Security
* IAM roles use least privilege access
* SNS topics are encrypted with AWS managed KMS keys
* Lambda functions run in default VPC

### 3. Maintenance
* CloudWatch Log retention: 14 days
* Lambda execution: Every 5 minutes
* Automatic alarm cleanup

### 4. Scaling
* Lambda timeout: 300 seconds
* Lambda memory: 256 MB
* Adjust based on resource count

## Best Practices
* Test in non-production account first
* Verify email subscriptions
* Monitor Lambda execution times
* Review CloudWatch Logs regularly
* Maintain consistent tagging strategy

## Limitations
* Maximum resources per account (adjust Lambda timeout if needed)
* AWS API rate limits
* CloudWatch alarm limits
* SNS topic subscription limits

## Support

### Troubleshooting Steps
* Check CloudWatch Logs
* Verify resource tagging
* Review IAM permissions
* Check Lambda configuration

### Common Issues
* IAM role permissions
* Resource tag mismatches
* Lambda timeouts
* SNS subscription confirmation

### Logs and Monitoring
* CloudWatch Log groups
* Lambda execution metrics
* CloudWatch Alarms
* SNS delivery status
```

