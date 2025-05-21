# AWS Ground Station JPSS-2 S3 Data Delivery Solution

This repository contains CloudFormation templates and documentation for setting up an AWS Ground Station mission profile for the JPSS-2 satellite with S3 data delivery in the AWS Ohio (us-east-2) region.

## Solution Overview

This solution enables automated downlink of data from the JPSS-2 satellite directly to an S3 bucket. The solution includes:

- Ground Station mission profile configuration
- S3 bucket for data storage
- IAM roles and permissions
- Network configuration (VPC, subnet, security group)
- Tracking, downlink, demodulation, and decoding configurations

![Architecture Diagram](https://via.placeholder.com/800x400?text=AWS+Ground+Station+JPSS-2+Architecture)

## Prerequisites

Before deploying this solution, you need:

1. An AWS account with access to AWS Ground Station service
2. AWS Ground Station onboarding completed
3. JPSS-2 satellite registered with your AWS Ground Station account
4. A VPC with at least one subnet in the us-east-2 (Ohio) region
5. AWS CLI installed and configured
6. Permissions to create CloudFormation stacks, IAM roles, S3 buckets, and Ground Station resources

## Deployment Instructions

### 1. Clone this repository

```bash
git clone https://github.com/yourusername/aws-groundstation-jpss2.git
cd aws-groundstation-jpss2
```

### 2. Review and customize parameters

Open `jpss2-ground-station-config.yaml` and review the default parameters. You may want to customize:

- `S3BucketName`: Name for your data delivery bucket
- `S3BucketPrefix`: Prefix for organizing data in the bucket
- `DownlinkCenterFrequency`: Center frequency in MHz (default: 8212.5)

### 3. Deploy the CloudFormation stack

```bash
aws cloudformation create-stack \
  --stack-name jpss2-ground-station \
  --template-body file://jpss2-ground-station-config.yaml \
  --parameters ParameterKey=VpcId,ParameterValue=vpc-12345678 \
               ParameterKey=SubnetId,ParameterValue=subnet-12345678 \
  --capabilities CAPABILITY_IAM \
  --region us-east-2
```

Replace `vpc-12345678` and `subnet-12345678` with your actual VPC and subnet IDs.

### 4. Monitor the deployment

```bash
aws cloudformation describe-stacks \
  --stack-name jpss2-ground-station \
  --region us-east-2
```

### 5. Get the outputs

Once the stack is deployed, retrieve the outputs:

```bash
aws cloudformation describe-stacks \
  --stack-name jpss2-ground-station \
  --query "Stacks[0].Outputs" \
  --region us-east-2
```

## Using the Solution

### Scheduling Contacts

After deployment, you can schedule contacts with the JPSS-2 satellite:

```bash
# List available contacts
aws groundstation list-contacts \
  --satellite-id <JPSS-2-SATELLITE-ID> \
  --status AVAILABLE \
  --region us-east-2

# Schedule a contact
aws groundstation reserve-contact \
  --contact-id <CONTACT-ID> \
  --mission-profile-arn <MISSION-PROFILE-ARN> \
  --region us-east-2
```

Replace `<JPSS-2-SATELLITE-ID>`, `<CONTACT-ID>`, and `<MISSION-PROFILE-ARN>` with your actual values. The mission profile ARN is available in the CloudFormation stack outputs.

### Accessing Downlinked Data

Data from the satellite will be delivered to the S3 bucket specified during deployment:

```bash
# List objects in the S3 bucket
aws s3 ls s3://<S3-BUCKET-NAME>/<S3-BUCKET-PREFIX>/ --region us-east-2

# Download a specific file
aws s3 cp s3://<S3-BUCKET-NAME>/<S3-BUCKET-PREFIX>/filename.dat ./local-filename.dat --region us-east-2
```

Replace `<S3-BUCKET-NAME>` and `<S3-BUCKET-PREFIX>` with the values from your deployment.

## Monitoring

### Contact Status

Monitor the status of scheduled contacts:

```bash
aws groundstation list-contacts \
  --satellite-id <JPSS-2-SATELLITE-ID> \
  --status SCHEDULED \
  --region us-east-2
```

### CloudWatch Metrics

AWS Ground Station publishes metrics to CloudWatch. You can create dashboards to monitor:

- Contact success rates
- Signal quality
- Data delivery status

## Troubleshooting

### Common Issues

1. **Contact Scheduling Failures**
   - Verify that the satellite is registered with AWS Ground Station
   - Check that the mission profile ARN is correct
   - Ensure the contact is within the satellite's visibility window

2. **Data Delivery Issues**
   - Check IAM role permissions
   - Verify S3 bucket exists and is accessible
   - Review CloudWatch logs for data delivery errors

3. **Signal Reception Problems**
   - Verify tracking configuration
   - Check downlink frequency settings
   - Review demodulation and decoding parameters

### Support

For issues with this solution:
- Open an issue in this GitHub repository
- Contact AWS Support for Ground Station service issues

## Cost Considerations

This solution incurs costs for:
- AWS Ground Station antenna time
- S3 storage for downlinked data
- Data transfer
- CloudWatch logs and metrics

Review the [AWS Ground Station pricing](https://aws.amazon.com/ground-station/pricing/) and [S3 pricing](https://aws.amazon.com/s3/pricing/) for details.

## Security

This solution creates IAM roles with specific permissions. Review these permissions to ensure they align with your security requirements.

The S3 bucket is created with default settings. Consider enabling:
- Server-side encryption
- Access logging
- Versioning
- Object lifecycle policies

## License

This project is licensed under the MIT License - see the LICENSE file for details.
