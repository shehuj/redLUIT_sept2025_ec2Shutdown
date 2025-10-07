# redLUIT_sept2025_ec2Shutdown
this repository holds the logic and pipeline for unused resources in AWS and shuts down basically unused instances (EC2)

# EC2 Automatic Shutdown Solution - FOUNDATIONAL.

## Overview

This repository contains a serverless automation solution for LevelUp Bank that automatically shuts down development EC2 instances after business hours to reduce AWS costs. The solution uses AWS Lambda, CloudFormation, and GitHub Actions to provide a fully automated CI/CD pipeline with separate beta and production environments.

## Architecture

- **AWS Lambda**: Serverless function that stops running EC2 instances
- **CloudFormation**: Infrastructure as Code for deploying Lambda, IAM roles, and EventBridge rules
- **EventBridge**: Scheduled trigger that runs the Lambda function daily at 7:00 PM
- **GitHub Actions**: CI/CD pipeline with environment separation (beta/prod)
- **S3**: Storage for Lambda deployment packages

## Repository Structure

```
.
├── lambda_function.py                              # Python Lambda function code
├── infrastructure/
│   └── cloudformation/
│       └── lambda-ec2-shutdown-foundational.yml                 # CloudFormation template
├── .github/
│   └── workflows/
│       ├── on_pull_request.yml                     # Beta deployment workflow
│       └── on_merge.yml                            # Production deployment workflow
└── README.md                                       # This file
```

## Prerequisites

1. **AWS Account** with appropriate permissions:
   - EC2 (describe and stop instances)
   - Lambda (create and invoke functions)
   - CloudFormation (create and update stacks)
   - IAM (create roles and policies)
   - S3 (upload Lambda packages)
   - EventBridge (create scheduled rules)

2. **S3 Buckets** for Lambda deployment packages:
   - One bucket for beta environment
   - One bucket for production environment

3. **GitHub Repository** with Actions enabled

## Setup Instructions

### Step 1: Download the CloudFormation Template

The CloudFormation template is provided in this Gist:
https://gist.github.com/zaireali649/af536f756a1b90e65975fe5b810f4f00

Save it to: `infrastructure/cloudformation/lambda-ec2-shutdown.yml`

### Step 2: Configure GitHub Secrets

Add the following secrets to your GitHub repository (Settings → Secrets and variables → Actions):

**AWS Credentials:**
- `AWS_ACCESS_KEY_ID` - AWS access key with required permissions
- `AWS_SECRET_ACCESS_KEY` - AWS secret access key
- `AWS_REGION` - AWS region (e.g., us-east-1)

**Beta Environment:**
- `CF_STACK_NAME_BETA` - CloudFormation stack name (e.g., ec2-shutdown-beta)
- `LAMBDA_NAME_BETA` - Lambda function name (e.g., ec2-shutdown-lambda-beta)
- `S3_BUCKET_BETA` - S3 bucket for beta deployments (e.g., levelup-lambda-beta)
- `S3_PATH_BETA` - S3 object key path (e.g., lambda/ec2-shutdown/function.zip)

**Production Environment:**
- `CF_STACK_NAME_PROD` - CloudFormation stack name (e.g., ec2-shutdown-prod)
- `LAMBDA_NAME_PROD` - Lambda function name (e.g., ec2-shutdown-lambda-prod)
- `S3_BUCKET_PROD` - S3 bucket for production deployments (e.g., levelup-lambda-prod)
- `S3_PATH_PROD` - S3 object key path (e.g., lambda/ec2-shutdown/function.zip)

### Step 3: Create S3 Buckets

Create the S3 buckets if they don't already exist:

```bash
aws s3 mb s3://levelup-lambda-beta --region us-east-1
aws s3 mb s3://levelup-lambda-prod --region us-east-1
```

### Step 4: Place Files in Repository

1. Copy `lambda_function.py` to the root of your repository
2. Copy the CloudFormation template to `infrastructure/cloudformation/lambda-ec2-shutdown.yml`
3. Copy the workflow files to `.github/workflows/`

## How It Works

### Lambda Function Logic

The `lambda_function_foundational.py` script:

1. Connects to EC2 using boto3
2. Queries for all instances with state `running`
3. Extracts instance IDs and logs their names
4. Stops all running instances
5. Returns a success/failure response with details

### CI/CD Pipeline

#### Beta Deployment (Pull Request)

When you create a pull request to `main`:

1. GitHub Actions triggers `on_pull_request.yml`
2. Lambda code is packaged into `function.zip`
3. Package is uploaded to the beta S3 bucket
4. CloudFormation deploys/updates the beta stack
5. A comment is added to the PR confirming successful deployment

**To trigger beta deployment:**
```bash
git checkout -b feature/update-lambda
# Make changes to lambda_function.py
git add .
git commit -m "Update Lambda logic"
git push origin feature/update-lambda
# Create pull request on GitHub
```

#### Production Deployment (Merge to Main)

When the pull request is merged to `main`:

1. GitHub Actions triggers `on_merge.yml`
2. Lambda code is packaged into `function.zip`
3. Package is uploaded to the production S3 bucket
4. CloudFormation deploys/updates the production stack
5. Deployment notification is logged

**To trigger production deployment:**
```bash
# After PR review and approval
# Merge the pull request on GitHub
```

### CloudFormation Stack Components

The CloudFormation template creates:

1. **Lambda Function**: Python 3.12 runtime, configured with S3 code location
2. **IAM Role**: Grants Lambda permissions to:
   - Describe EC2 instances
   - Stop EC2 instances
   - Write CloudWatch logs
3. **EventBridge Rule**: Scheduled trigger (cron: 0 19 * * ? *) for 7:00 PM UTC daily
4. **Lambda Permission**: Allows EventBridge to invoke the Lambda function

## Modifying the Lambda Logic

### Filter by Tags

To stop only instances with specific tags (e.g., Environment=dev):

```python
response = ec2.describe_instances(
    Filters=[
        {
            'Name': 'instance-state-name',
            'Values': ['running']
        },
        {
            'Name': 'tag:Environment',
            'Values': ['dev']
        }
    ]
)
```

### Filter by Instance Type

To stop only specific instance types:

```python
response = ec2.describe_instances(
    Filters=[
        {
            'Name': 'instance-state-name',
            'Values': ['running']
        },
        {
            'Name': 'instance-type',
            'Values': ['t2.micro', 't3.small']
        }
    ]
)
```

### Exclude Production Instances

To avoid stopping instances tagged as production:

```python
instances_to_stop = []
for reservation in response['Reservations']:
    for instance in reservation['Instances']:
        # Check if instance has Environment=prod tag
        is_production = False
        if 'Tags' in instance:
            for tag in instance['Tags']:
                if tag['Key'] == 'Environment' and tag['Value'] == 'prod':
                    is_production = True
                    break
        
        if not is_production:
            instances_to_stop.append(instance['InstanceId'])
```

### Change the Schedule

Edit the CloudFormation template to change the EventBridge schedule:

```yaml
ScheduleExpression: "cron(0 21 * * ? *)"  # 9:00 PM UTC
```

Cron format: `cron(minute hour day-of-month month day-of-week year)`

## Verification and Testing

### Test Lambda Function Manually

1. Go to AWS Lambda Console
2. Select your Lambda function (beta or prod)
3. Click "Test" and create a test event (empty JSON: `{}`)
4. Click "Test" to execute
5. Review the execution results and logs

### View CloudWatch Logs

1. Go to CloudWatch Console → Log groups
2. Find log group: `/aws/lambda/your-function-name`
3. View recent log streams to see execution details

### Check EC2 Instance Status

```bash
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].[InstanceId,State.Name,Tags[?Key==`Name`].Value|[0]]' \
  --output table
```

### Monitor GitHub Actions

1. Go to your repository → Actions tab
2. View workflow runs for each deployment
3. Check logs for each step to debug issues

### Verify Stack Deployment

```bash
# Check beta stack
aws cloudformation describe-stacks --stack-name ec2-shutdown-beta

# Check production stack
aws cloudformation describe-stacks --stack-name ec2-shutdown-prod
```

## Troubleshooting

### Lambda Function Not Stopping Instances

1. Check CloudWatch logs for error messages
2. Verify IAM role has `ec2:DescribeInstances` and `ec2:StopInstances` permissions
3. Ensure Lambda is in the same region as EC2 instances
4. Test manually in Lambda console

### GitHub Actions Deployment Fails

1. Verify all GitHub secrets are configured correctly
2. Check AWS credentials have necessary permissions
3. Ensure S3 buckets exist and are accessible
4. Review CloudFormation stack events for errors

### EventBridge Not Triggering Lambda

1. Verify EventBridge rule is enabled
2. Check rule's schedule expression
3. Ensure Lambda has permission for EventBridge to invoke it
4. Review CloudWatch metrics for Lambda invocations

## Security Best Practices

1. **Use IAM roles with least privilege**: Only grant permissions necessary for the function
2. **Never commit AWS credentials**: Always use GitHub Secrets
3. **Enable CloudTrail**: Monitor all API calls for audit purposes
4. **Use VPC endpoints**: For enhanced security, deploy Lambda in a VPC
5. **Tag resources**: Apply consistent tagging for better access control
6. **Regular reviews**: Periodically review IAM policies and Lambda logs

## Cost Optimization

- **Lambda**: Pay only for execution time (typically < $1/month for daily runs)
- **EventBridge**: No additional cost for scheduled rules
- **S3**: Minimal storage costs for deployment packages
- **CloudWatch Logs**: Set retention period to 7-30 days to reduce costs

**Estimated monthly cost**: Less than $5 for the entire solution

## Additional Resources

- [AWS Lambda Documentation](https://docs.aws.amazon.com/lambda/)
- [Boto3 EC2 Documentation](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/ec2.html)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [CloudFormation Documentation](https://docs.aws.amazon.com/cloudformation/)

## Support

For issues or questions:
1. Check CloudWatch logs for Lambda execution errors
2. Review GitHub Actions workflow logs
3. Consult AWS CloudFormation stack events
4. Contact your DevOps team for assistance