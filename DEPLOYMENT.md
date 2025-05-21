# Detailed Deployment Guide

This document provides detailed instructions for deploying the AWS Ground Station JPSS-2 S3 Data Delivery Solution.

## Pre-deployment Checklist

Before proceeding with deployment, ensure you have:

- [ ] Completed AWS Ground Station onboarding
- [ ] Registered JPSS-2 satellite with your AWS Ground Station account
- [ ] Created a VPC and subnet in the us-east-2 region
- [ ] Installed and configured AWS CLI with appropriate permissions
- [ ] Reviewed the CloudFormation template parameters

## Step-by-Step Deployment

### 1. Prepare Your Environment

Ensure your AWS CLI is configured for the us-east-2 region:

```bash
aws configure set region us-east-2
```

Verify your AWS CLI can access the required services:

```bash
aws groundstation get-config --region us-east-2
aws s3 ls --region us-east-2
aws cloudformation describe-stacks --region us-east-2
```

### 2. Customize the CloudFormation Template

The template includes several parameters that you may want to customize:

| Parameter | Description | Default | Recommendation |
|-----------|-------------|---------|---------------|
| S3BucketName | Name of the S3 bucket for data delivery | jpss2-ground-station-data | Use a globally unique name |
| S3BucketPrefix | Prefix for S3 objects | jpss2-data/ | Include date patterns for better organization |
| VpcId | VPC ID for Ground Station endpoint | - | Use a VPC with internet access |
| SubnetId | Subnet ID for Ground Station endpoint | - | Use a subnet with internet access |
| DownlinkCenterFrequency | Center frequency in MHz | 8212.5 | Verify with JPSS-2 specifications |

### 3. Validate the CloudFormation Template

Before deployment, validate the template:

```bash
aws cloudformation validate-template \
  --template-body file://jpss2-ground-station-config.yaml \
  --region us-east-2
```

### 4. Deploy the CloudFormation Stack

Deploy the stack with your customized parameters:

```bash
aws cloudformation create-stack \
  --stack-name jpss2-ground-station \
  --template-body file://jpss2-ground-station-config.yaml \
  --parameters \
      ParameterKey=S3BucketName,ParameterValue=your-unique-bucket-name \
      ParameterKey=S3BucketPrefix,ParameterValue=jpss2-data/ \
      ParameterKey=VpcId,ParameterValue=vpc-12345678 \
      ParameterKey=SubnetId,ParameterValue=subnet-12345678 \
      ParameterKey=DownlinkCenterFrequency,ParameterValue=8212.5 \
  --capabilities CAPABILITY_IAM \
  --region us-east-2
```

### 5. Monitor Deployment Progress

Check the status of your deployment:

```bash
aws cloudformation describe-stack-events \
  --stack-name jpss2-ground-station \
  --region us-east-2
```

### 6. Retrieve Stack Outputs

Once the stack is created successfully, retrieve the outputs:

```bash
aws cloudformation describe-stacks \
  --stack-name jpss2-ground-station \
  --query "Stacks[0].Outputs" \
  --output table \
  --region us-east-2
```

Save these outputs for future reference, especially the MissionProfileArn which you'll need for scheduling contacts.

## Post-Deployment Configuration

### 1. Verify Resource Creation

Verify that all resources were created correctly:

```bash
# Check S3 bucket
aws s3 ls s3://your-unique-bucket-name --region us-east-2

# Check Ground Station configs
aws groundstation list-configs --region us-east-2

# Check Mission Profile
aws groundstation list-mission-profiles --region us-east-2
```

### 2. Configure S3 Bucket Security (Optional)

For enhanced security, consider enabling server-side encryption:

```bash
aws s3api put-bucket-encryption \
  --bucket your-unique-bucket-name \
  --server-side-encryption-configuration '{
    "Rules": [
      {
        "ApplyServerSideEncryptionByDefault": {
          "SSEAlgorithm": "AES256"
        }
      }
    ]
  }' \
  --region us-east-2
```

### 3. Set Up S3 Lifecycle Policies (Optional)

If you expect large volumes of data, consider setting up lifecycle policies:

```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket your-unique-bucket-name \
  --lifecycle-configuration '{
    "Rules": [
      {
        "ID": "ArchiveOldData",
        "Status": "Enabled",
        "Prefix": "jpss2-data/",
        "Transitions": [
          {
            "Days": 90,
            "StorageClass": "GLACIER"
          }
        ]
      }
    ]
  }' \
  --region us-east-2
```

## Troubleshooting Deployment Issues

### Common Deployment Errors

1. **S3 Bucket Name Already Exists**
   - Error: "Error creating resource, bucket already exists"
   - Solution: Choose a different, globally unique S3 bucket name

2. **Insufficient Permissions**
   - Error: "User is not authorized to perform action on resource"
   - Solution: Ensure your IAM user/role has the necessary permissions

3. **VPC/Subnet Configuration Issues**
   - Error: "The subnet ID 'subnet-xxx' does not exist"
   - Solution: Verify the subnet ID exists in the specified region

4. **IAM Role Creation Failure**
   - Error: "Requires capabilities : [CAPABILITY_IAM]"
   - Solution: Include the `--capabilities CAPABILITY_IAM` flag in your deployment command

### Rollback and Cleanup

If you need to remove the deployment:

```bash
aws cloudformation delete-stack \
  --stack-name jpss2-ground-station \
  --region us-east-2
```

Note: This will delete all resources created by the stack except for the S3 bucket if it contains objects. To delete a non-empty S3 bucket:

```bash
aws s3 rm s3://your-unique-bucket-name --recursive
aws s3api delete-bucket --bucket your-unique-bucket-name --region us-east-2
```
