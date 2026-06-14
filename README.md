# AI Knowledge Assistant — AWS Bedrock RAG

> Serverless AI assistant built on AWS Bedrock Knowledge Base (RAG). Ask questions in natural language about your documents via voice.

---

## Overview

This project implements a fully serverless RAG (Retrieval-Augmented Generation) architecture on AWS, enabling natural language queries over private documents. The assistant is accessible via a voice interface.

**Use cases:**
- Personal home assistant (equipment references, procedures, technical docs)
- Knowledge base 

---

## Architecture

```
User (Further Devices)
        │
        │ HTTPS POST /chat
        ▼
API Gateway (REST)
        │
        │ Invoke
        ▼
Lambda (Python 3.12)
        │
        │ RetrieveAndGenerate
        ▼
Bedrock Knowledge Base ──── Vector Search ──── S3 Vectors
        │
        │ Document Retrieval
        ▼
S3 (Source Documents)

── Observability ──
Lambda → CloudWatch Logs (30 days retention)

── Security ──
IAM Roles (Least Privilege)
  └── Lambda Execution Role
  └── Bedrock Knowledge Base Role

── IaC ──
CloudFormation Stack
```

---

## AWS Services

| Service | Role |
|---------|------|
| **API Gateway** | REST API exposure — POST /chat |
| **Lambda** | Business logic — Python 3.12, 256MB, 30s timeout |
| **Bedrock Knowledge Base** | RAG orchestration |
| **Claude Haiku 4.5** | LLM for answer generation |
| **Titan Text Embeddings v2** | Document vectorization |
| **S3 Vectors** | Vector store (embeddings) |
| **S3** | Source document storage |
| **CloudWatch Logs** | Lambda observability |
| **IAM** | Least privilege roles |
| **CloudFormation** | Infrastructure as Code |

---

## Key Design Decisions

- **Serverless** — No EC2, auto-scaling, pay-per-use
- **eu-west-3 (Paris)** — Data sovereignty, GDPR compliance
- **S3 Vectors** — Lightweight vector store, no OpenSearch overhead
- **Claude Haiku 4.5** — Optimal cost/performance ratio for Q&A
- **No KMS** — Data is non-sensitive, default AWS encryption sufficient
- **CloudFormation** — Full IaC, reproducible deployment

---

## Cost Estimate

| Service | Monthly Cost |
|---------|-------------|
| API Gateway | ~$0.01 |
| Lambda | ~$0.01 |
| Bedrock (Claude Haiku 4.5) | ~$0.24 |
| S3 | ~$0.01 |
| CloudWatch | ~$0.01 |
| **Total** | **~$0.30/month** |

---

## Deployment

### Prerequisites
- AWS CLI configured
- AWS account with Bedrock model access (Claude Haiku 4.5)

### Deploy the stack

```bash
aws cloudformation deploy \
  --template-file maison-chatbot.yaml \
  --stack-name maison-chatbot \
  --region eu-west-3 \
  --capabilities CAPABILITY_NAMED_IAM
```

### Upload documents to S3

```bash
aws s3 cp your-document.docx \
  s3://$(aws cloudformation describe-stacks \
    --stack-name maison-chatbot \
    --region eu-west-3 \
    --query 'Stacks[0].Outputs[?OutputKey==`DocumentsBucketName`].OutputValue' \
    --output text)/
```

### Create Bedrock Knowledge Base
1. AWS Console → Amazon Bedrock → Knowledge Bases → Create
2. Select S3 bucket as data source
3. Choose S3 Vectors as vector store
4. Select Titan Text Embeddings v2

### Update Lambda with Knowledge Base ID

```bash
aws lambda update-function-configuration \
  --function-name maison-chatbot-chatbot \
  --region eu-west-3 \
  --environment "Variables={KNOWLEDGE_BASE_ID=YOUR_KB_ID,MODEL_ID=eu.anthropic.claude-haiku-4-5-20251001-v1:0}"
```

### Sync documents

```bash
aws bedrock-agent start-ingestion-job \
  --knowledge-base-id YOUR_KB_ID \
  --data-source-id YOUR_DS_ID \
  --region eu-west-3
```

### Test the API

```bash
curl -X POST https://YOUR_API_ID.execute-api.eu-west-3.amazonaws.com/prod/chat \
  -H "Content-Type: application/json" \
  -d '{"question": "What is the circuit breaker for the bathroom?"}'
```

---


## Repository Structure

```
├── maison-chatbot.yaml          # CloudFormation template
├── assistant-maison.html        # Web interface (Chrome)
├── architecture-assistant-maison.drawio  # Architecture diagram
└── README.md
```

---

## Author

**Jérémie Maumet**
Cloud Operations Engineer | AWS SAA-C03 ✓ | AZ-305 in progress

[LinkedIn](https://linkedin.com/in/jeremie-maumet) · [GitHub](https://github.com/jeremie-maumet)
