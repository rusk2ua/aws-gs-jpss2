# Using the AWS Ground Station JPSS-2 Solution

This guide provides detailed instructions for using the AWS Ground Station JPSS-2 S3 Data Delivery Solution after deployment.

## Scheduling Contacts

### Finding Available Contacts

To find available contacts with the JPSS-2 satellite:

```bash
aws groundstation list-contacts \
  --satellite-id <JPSS-2-SATELLITE-ID> \
  --status AVAILABLE \
  --start-time $(date -u +"%Y-%m-%dT%H:%M:%SZ") \
  --end-time $(date -u -d "+7 days" +"%Y-%m-%dT%H:%M:%SZ") \
  --region us-east-2
```

This command lists all available contacts for the next 7 days. You can adjust the time range as needed.

### Scheduling a Contact

Once you've identified a contact you want to schedule:

```bash
aws groundstation reserve-contact \
  --contact-id <CONTACT-ID> \
  --mission-profile-arn <MISSION-PROFILE-ARN> \
  --region us-east-2
```

Replace:
- `<CONTACT-ID>` with the ID of the contact you want to schedule
- `<MISSION-PROFILE-ARN>` with the ARN of the mission profile created by the CloudFormation stack

### Viewing Scheduled Contacts

To view your scheduled contacts:

```bash
aws groundstation list-contacts \
  --satellite-id <JPSS-2-SATELLITE-ID> \
  --status SCHEDULED \
  --region us-east-2
```

### Canceling a Contact

If you need to cancel a scheduled contact:

```bash
aws groundstation cancel-contact \
  --contact-id <CONTACT-ID> \
  --region us-east-2
```

## Managing Data

### Viewing Downlinked Data

After a successful contact, data will be delivered to your S3 bucket. To list the objects:

```bash
aws s3 ls s3://<S3-BUCKET-NAME>/<S3-BUCKET-PREFIX>/ --region us-east-2
```

The data files are typically organized by contact ID and timestamp.

### Downloading Data

To download a specific file:

```bash
aws s3 cp s3://<S3-BUCKET-NAME>/<S3-BUCKET-PREFIX>/path/to/file.dat ./local-file.dat --region us-east-2
```

To download all data from a specific contact:

```bash
aws s3 sync s3://<S3-BUCKET-NAME>/<S3-BUCKET-PREFIX>/contact-<CONTACT-ID>/ ./local-folder/ --region us-east-2
```

### Data Format

The data is delivered in its raw format as received from the satellite. Depending on the JPSS-2 data format, you may need additional processing tools to extract useful information.

## Monitoring

### Contact Status

To monitor the status of a specific contact:

```bash
aws groundstation describe-contact \
  --contact-id <CONTACT-ID> \
  --region us-east-2
```

### CloudWatch Metrics

AWS Ground Station publishes metrics to CloudWatch. To view these metrics:

1. Open the CloudWatch console: https://console.aws.amazon.com/cloudwatch/
2. Navigate to Metrics > All metrics
3. Select the "AWS/GroundStation" namespace
4. View metrics by MissionProfileId, ContactId, or other dimensions

Key metrics to monitor:
- `ContactSuccessful` - Whether the contact was successful
- `ContactFailed` - Whether the contact failed
- `BytesTransferred` - Amount of data transferred

### Setting Up CloudWatch Alarms

You can set up alarms to notify you of contact failures:

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "JPSS2-Contact-Failure-Alarm" \
  --alarm-description "Alarm when a JPSS-2 contact fails" \
  --metric-name "ContactFailed" \
  --namespace "AWS/GroundStation" \
  --statistic "Sum" \
  --period 60 \
  --threshold 1 \
  --comparison-operator "GreaterThanOrEqualToThreshold" \
  --dimensions Name=MissionProfileId,Value=<MISSION-PROFILE-ID> \
  --evaluation-periods 1 \
  --alarm-actions <SNS-TOPIC-ARN> \
  --region us-east-2
```

Replace:
- `<MISSION-PROFILE-ID>` with the ID portion of your mission profile ARN
- `<SNS-TOPIC-ARN>` with the ARN of an SNS topic for notifications

## Advanced Usage

### Automating Contact Scheduling

You can create a script to automatically schedule contacts:

```bash
#!/bin/bash

SATELLITE_ID="<JPSS-2-SATELLITE-ID>"
MISSION_PROFILE_ARN="<MISSION-PROFILE-ARN>"
REGION="us-east-2"

# Get available contacts for the next 24 hours
START_TIME=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
END_TIME=$(date -u -d "+24 hours" +"%Y-%m-%dT%H:%M:%SZ")

CONTACTS=$(aws groundstation list-contacts \
  --satellite-id $SATELLITE_ID \
  --status AVAILABLE \
  --start-time $START_TIME \
  --end-time $END_TIME \
  --region $REGION \
  --query "contactList[*].contactId" \
  --output text)

# Schedule each available contact
for CONTACT_ID in $CONTACTS; do
  echo "Scheduling contact: $CONTACT_ID"
  aws groundstation reserve-contact \
    --contact-id $CONTACT_ID \
    --mission-profile-arn $MISSION_PROFILE_ARN \
    --region $REGION
done
```

### Data Processing Pipeline

For automated data processing, consider setting up an S3 event notification to trigger an AWS Lambda function when new data is delivered:

1. Create a Lambda function to process the data
2. Configure an S3 event notification:

```bash
aws s3api put-bucket-notification-configuration \
  --bucket <S3-BUCKET-NAME> \
  --notification-configuration '{
    "LambdaFunctionConfigurations": [
      {
        "LambdaFunctionArn": "<LAMBDA-FUNCTION-ARN>",
        "Events": ["s3:ObjectCreated:*"],
        "Filter": {
          "Key": {
            "FilterRules": [
              {
                "Name": "prefix",
                "Value": "<S3-BUCKET-PREFIX>/"
              }
            ]
          }
        }
      }
    ]
  }' \
  --region us-east-2
```

## Troubleshooting

### Contact Failures

If a contact fails:

1. Check the CloudWatch logs for the contact
2. Verify that the satellite was visible from the ground station
3. Check that the tracking, demodulation, and decoding configurations match the satellite's specifications
4. Ensure the IAM role has the necessary permissions

### Data Delivery Issues

If data is not delivered to S3:

1. Check the IAM role permissions
2. Verify the S3 bucket exists and is accessible
3. Check the dataflow endpoint configuration
4. Review CloudWatch logs for data delivery errors

### Signal Quality Issues

If the signal quality is poor:

1. Verify the tracking configuration
2. Check the downlink frequency settings
3. Review the demodulation and decoding parameters
4. Consider weather conditions at the ground station during the contact

## Support Resources

- [AWS Ground Station Documentation](https://docs.aws.amazon.com/ground-station/)
- [AWS Ground Station API Reference](https://docs.aws.amazon.com/ground-station/latest/APIReference/)
- [AWS Ground Station User Guide](https://docs.aws.amazon.com/ground-station/latest/ug/)
