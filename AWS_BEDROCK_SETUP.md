# AWS Bedrock Setup Guide for Secludy AutoPilot Redactor

This guide walks you through setting up AWS Bedrock to use Claude Opus 4.1 for PII pattern discovery.

## Prerequisites

- AWS account with programmatic access configured
- AWS CLI version 2.13.23 or newer
- Python 3.9+

## 1. Install and Configure AWS CLI

### Install AWS CLI
```bash
# Linux
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Verify installation
aws --version
```

### Configure AWS Credentials
```bash
# Option 1: Interactive configuration
aws configure

# Option 2: Export environment variables
export AWS_ACCESS_KEY_ID=your-access-key
export AWS_SECRET_ACCESS_KEY=your-secret-key
export AWS_REGION=us-west-2

# Verify credentials
aws sts get-caller-identity
```

## 2. Enable Claude Models in Bedrock

1. Go to [AWS Console > Bedrock > Model Access](https://console.aws.amazon.com/bedrock/home?region=us-west-2#/modelaccess)
2. Request access to Anthropic models
3. Wait for approval (usually instant for Claude models)

### Available Claude Models

| Model | Bedrock API Model Name |
|-------|------------------------|
| **Claude Opus 4.1** ⭐ | `anthropic.claude-opus-4-1-20250805-v1:0` |
| Claude Opus 4 | `anthropic.claude-opus-4-20250514-v1:0` |
| Claude Sonnet 4 | `anthropic.claude-sonnet-4-20250514-v1:0` |

### Check Model Availability
```bash
# List all available Claude models in your region
aws bedrock list-foundation-models \
  --region=us-west-2 \
  --by-provider anthropic \
  --query "modelSummaries[*].modelId"
```

## 3. Install SDK

### For Python (Recommended for Secludy)
```bash
pip install -U "anthropic[bedrock]"
# OR using boto3 directly
pip install boto3>=1.28.59
```

## 4. Test Bedrock Connection

### Python Test Script
```python
from anthropic import AnthropicBedrock

# Initialize client
client = AnthropicBedrock(
    # Uses AWS credentials from environment or ~/.aws/credentials
    aws_region="us-west-2",
)

# Test API call
message = client.messages.create(
    model="anthropic.claude-opus-4-1-20250805-v1:0",
    max_tokens=256,
    messages=[{"role": "user", "content": "Hello, Claude on Bedrock!"}]
)
print(message.content)
```

### Using Boto3 (Alternative)
```python
import boto3
import json

bedrock = boto3.client(
    service_name="bedrock-runtime",
    region_name="us-west-2"
)

body = json.dumps({
    "max_tokens": 256,
    "messages": [{"role": "user", "content": "Hello, Claude on Bedrock!"}],
    "anthropic_version": "bedrock-2023-05-31"
})

response = bedrock.invoke_model(
    body=body, 
    modelId="anthropic.claude-opus-4-1-20250805-v1:0"
)

response_body = json.loads(response.get("body").read())
print(response_body.get("content"))
```

## 5. Configure Secludy for Bedrock

### Environment Variables (.env)
```bash
# AWS Credentials
AWS_ACCESS_KEY_ID=your-access-key
AWS_SECRET_ACCESS_KEY=your-secret-key
AWS_REGION=us-west-2

# Bedrock Configuration
LLM_PROVIDER=bedrock
BEDROCK_MODEL=anthropic.claude-opus-4-1-20250805-v1:0
BEDROCK_MAX_TOKENS=4096
BEDROCK_TEMPERATURE=0.3
```

### Docker Compose Configuration
```yaml
services:
  rule-generator:
    image: secludy/autopilot-redactor:latest
    environment:
      - LLM_PROVIDER=bedrock
      - AWS_REGION=${AWS_REGION:-us-west-2}
      - AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
      - AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
      - BEDROCK_MODEL=anthropic.claude-opus-4-1-20250805-v1:0
```

### Helm Values Configuration
```yaml
aws:
  region: us-west-2
  credentials:
    # Option 1: Create secret manually
    existingSecret: "aws-bedrock-credentials"
    # Option 2: Provide in values (not recommended for production)
    accessKeyId: ""
    secretAccessKey: ""

discovery:
  llmProvider: bedrock
  bedrockModel: anthropic.claude-opus-4-1-20250805-v1:0
```

## 6. IAM Permissions

Create an IAM policy with the following permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel",
        "bedrock:InvokeModelWithResponseStream"
      ],
      "Resource": [
        "arn:aws:bedrock:us-west-2::foundation-model/anthropic.claude-opus-4-1*",
        "arn:aws:bedrock:us-west-2::foundation-model/anthropic.claude-opus-4*",
        "arn:aws:bedrock:us-west-2::foundation-model/anthropic.claude-sonnet*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "bedrock:ListFoundationModels",
        "bedrock:GetFoundationModel"
      ],
      "Resource": "*"
    }
  ]
}
```

## 7. Enable Activity Logging

### Configure CloudWatch Logging
```bash
aws bedrock put-model-invocation-logging-configuration \
  --logging-config '{
    "cloudWatchConfig": {
      "logGroupName": "/aws/bedrock/secludy",
      "roleArn": "arn:aws:iam::account-id:role/BedrockLoggingRole"
    },
    "textDataDeliveryEnabled": true,
    "imageDataDeliveryEnabled": false,
    "embeddingDataDeliveryEnabled": false
  }'
```

### Create CloudWatch Dashboard
```python
import boto3

cloudwatch = boto3.client('cloudwatch')

# Create custom metrics for monitoring
cloudwatch.put_metric_data(
    Namespace='Secludy/Bedrock',
    MetricData=[
        {
            'MetricName': 'PIIDiscoveryRequests',
            'Value': 1,
            'Unit': 'Count',
            'Dimensions': [
                {'Name': 'Model', 'Value': 'claude-opus-4.1'},
                {'Name': 'Environment', 'Value': 'production'}
            ]
        }
    ]
)
```

## 8. Advanced Features

### 1M Token Context Window (Beta)
For processing large documents:
```python
client = AnthropicBedrock(aws_region="us-west-2")

# Include beta header for 1M context
message = client.messages.create(
    model="anthropic.claude-sonnet-4-20250514-v1:0",
    max_tokens=4096,
    messages=[{"role": "user", "content": large_document}],
    extra_headers={
        "anthropic-beta": "context-1m-2025-08-07"
    }
)
```

## 9. Troubleshooting

### Common Issues

| Issue | Solution |
|-------|----------|
| "Model not found" | Ensure model access is granted in Bedrock console |
| "Invalid credentials" | Verify AWS credentials with `aws sts get-caller-identity` |
| "Region not supported" | Check [AWS documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/models-regions.html) for model availability |
| "Rate limit exceeded" | Implement exponential backoff or request quota increase |
| "Timeout errors" | Increase timeout settings or use smaller batch sizes |

### Debug Logging
```python
import logging
import boto3

# Enable debug logging
boto3.set_stream_logger('boto3.resources', logging.DEBUG)
boto3.set_stream_logger('botocore', logging.DEBUG)

# Test connection
bedrock = boto3.client('bedrock-runtime', region_name='us-west-2')
print(f"Endpoint: {bedrock._endpoint}")
```

## 10. Production Best Practices

1. **Use IAM Roles** instead of access keys in production
2. **Enable CloudWatch logging** for audit trails
3. **Implement retry logic** with exponential backoff
4. **Use parameter store** for sensitive configuration
5. **Implement request caching** to reduce API calls
6. **Use batch processing** for large datasets

## 11. Quick Start Command

```bash
# One-liner to test Bedrock setup
aws bedrock-runtime invoke-model \
  --model-id anthropic.claude-opus-4-1-20250805-v1:0 \
  --body '{"max_tokens":256,"messages":[{"role":"user","content":"Hello"}],"anthropic_version":"bedrock-2023-05-31"}' \
  --region us-west-2 \
  output.json && cat output.json | jq -r '.content[0].text'
```

## Support Resources

- [AWS Bedrock Documentation](https://docs.aws.amazon.com/bedrock/)
- [Anthropic Bedrock Guide](https://docs.anthropic.com/en/docs/build-with-claude/aws-bedrock)
- [AWS Model Regions](https://docs.aws.amazon.com/bedrock/latest/userguide/models-regions.html)