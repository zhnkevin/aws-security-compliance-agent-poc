# Security & Compliance AI Agent (Proof of Concept)
![](https://raw.githubusercontent.com/zhnkevin/aws-interview-security-compliance-agent/refs/heads/main/aws_interview_agent_architecture.png)

Proof of concept AI Agent built using Strands SDK that has a security & compliance persona. 
- Deployed via a Lambda function that sends outbound API calls via API Gateway to Amazon Bedrock for accessing the LLM and the knowledge base.
- Includes a simple agent evaluator that scores tool selection and output of response from agent. Stores results as a JSON in an S3 bucket. 
- OpenTelemetry is used to send observability data to CloudWatch.
