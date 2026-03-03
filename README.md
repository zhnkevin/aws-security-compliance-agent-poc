# Security & Compliance AI Agent (Proof of Concept)
![](https://raw.githubusercontent.com/zhnkevin/aws-security-compliance-agent-poc/refs/heads/main/aws_agent_architecture.png)

# Deployment Guide

## Architecture Overview

This is a proof of concept AI agent that I developed to understand building out an AI agent using AWS services. 

This project consists of two Lambda functions:
- **Agent Lambda** (`agent.py`) — the main security & compliance chatbot
- **Evaluator Lambda** (`agent_evaluator.py`) — invokes the agent with test cases and stores results in S3

Both use Amazon Bedrock (Nova Micro model + Knowledge Base) and OpenTelemetry for observability via CloudWatch.

---

## Prerequisites

- AWS CLI configured with sufficient permissions
- Python 3.x (see `.python-version`)
- An S3 bucket for evaluation results
- A Bedrock Knowledge Base containing your SHIP PDF document
- Bedrock model access enabled for `us.amazon.nova-micro-v1:0` in your AWS region

---

## Step 1: Create a Bedrock Knowledge Base

1. In the Bedrock console, go to **Knowledge Bases** → **Create knowledge base**
2. Upload any PDFor text file to an S3 bucket and use it as the data source
3. Note the **Knowledge Base ID** — it will be needed by the `retrieve` tool at runtime. This can be added as an ENV Varible to your Lambda. 

---

## Step 2: Create an S3 Bucket for Evaluation Results

```bash
aws s3 mb s3://<your-evaluation-bucket-name>
```

---

## Step 3: Package Dependencies

Lambda requires dependencies to be bundled with the deployment package. You can add the included ZIP file as a Lambda layer. 

---

## Step 4: Create IAM Roles

### Agent Lambda Role

Attach the following policies:
- `AmazonBedrockFullAccess` (or a scoped policy allowing `bedrock:InvokeModel` and `bedrock:Retrieve`)
- `CloudWatchLogsFullAccess` (for OpenTelemetry/OTLP traces)

### Evaluator Lambda Role

Attach the following policies:
- `AmazonS3FullAccess` (or scoped to your evaluation bucket)
- Permission to invoke the Agent Lambda: `lambda:InvokeFunction`

---

## Step 5: Deploy the Agent Lambda

You will need to create a Lambda layer with the "StrandAgents-LambdaLayer.zip" file. Copy the ARN. 

Make sure the code is in a ZIP file. 

```bash
aws lambda create-function \
  --function-name security-compliance-agent \
  --runtime python3.12 \
  --layers "YOUR-LAYER-ARN"
  --role arn:aws:iam::<account-id>:role/<agent-lambda-role> \
  --handler agent.lambda_handler \
  --zip-file fileb://agent.zip \
  --timeout 300 \
  --memory-size 512
```

> The agent makes outbound HTTP calls to the AWS Knowledge MCP endpoint and Bedrock, so set timeout to at least **300 seconds**.

---

## Step 6: Deploy the Evaluator Lambda

Make sure the code is in a ZIP file. 

```bash
aws lambda create-function \
  --function-name security-compliance-evaluator \
  --runtime python3.12 \
  --role arn:aws:iam::<account-id>:role/<evaluator-lambda-role> \
  --handler agent_evaluator.lambda_handler \
  --zip-file fileb://evaluator.zip \
  --timeout 900 \
  --memory-size 256
```

> Set timeout to **900 seconds** since the evaluator invokes the agent multiple times sequentially across 8 test cases.

---

## Step 7: Invoke the Agent Lambda

Test the agent directly with:

```bash
aws lambda invoke \
  --function-name security-compliance-agent \
  --payload '{"prompt": "What are the most common compliance frameworks?", "user": {"session_id": "test-session-1"}}' \
  response.json

cat response.json
```

---

## Step 8: Run the Evaluator

```bash
aws lambda invoke \
  --function-name security-compliance-evaluator \
  --payload '{
    "agent_lambda_name": "security-compliance-agent",
    "s3_bucket": "<your-evaluation-bucket-name>",
    "s3_prefix": "agent-evaluations",
    "session_id": "eval-session-001"
  }' \
  eval_response.json

cat eval_response.json
```

Evaluation results (per-test scores and summary) are saved to:
```
s3://<your-evaluation-bucket-name>/agent-evaluations/results-<timestamp>.json
s3://<your-evaluation-bucket-name>/agent-evaluations/summary-<timestamp>.json
```

---

## Step 9: View Observability Data

OpenTelemetry traces are exported via OTLP. To view them in CloudWatch:

1. Go to **CloudWatch** → **X-Ray traces** or **Application Signals**
2. Filter by `session.id` to trace individual user sessions

---
