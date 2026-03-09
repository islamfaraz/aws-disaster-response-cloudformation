# 🆘 AWS Real-Time Disaster Response Network

> **A serverless, event-driven platform for real-time disaster incident reporting, emergency resource coordination, and public alert broadcasting — saving lives through faster response times.**

[![Build Status](https://dev.azure.com/your-org/disaster-response/_apis/build/status/disaster-response-ci?branchName=main)](https://dev.azure.com/your-org/disaster-response/_build)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![IaC: CloudFormation](https://img.shields.io/badge/IaC-CloudFormation-orange.svg)](https://aws.amazon.com/cloudformation/)
[![Cloud: AWS](https://img.shields.io/badge/Cloud-AWS-yellow.svg)](https://aws.amazon.com/)

---

## 🌍 Why This Project?

Natural disasters affect **350 million people annually**. Delayed emergency response costs lives. This platform enables:

- **Crowd-Sourced Incident Reporting** — Citizens report disasters (floods, earthquakes, fires) via mobile/web API in real-time
- **Automated Alert Broadcasting** — Critical incidents trigger SNS alerts to first responders, NGOs, and government agencies
- **Resource Coordination** — Track and allocate emergency vehicles, shelters, medical supplies, and volunteers
- **Media Evidence** — Upload photos/videos of incidents for damage assessment
- **Multi-Region Query** — Query incidents by severity, region, or time for situational awareness

---

## 🏗️ Architecture

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                     AWS Disaster Response Network                            │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌───────────┐     ┌──────────────┐     ┌────────────────┐                   │
│  │  Mobile    │────▶│  API Gateway │────▶│  Lambda        │                  │
│  │  Web App   │     │  (REST API)  │     │  (Python 3.12) │                  │
│  │  IoT       │     │  Throttled   │     │  3 Functions   │                  │
│  └───────────┘     └──────────────┘     └───────┬────────┘                  │
│                                                  │                           │
│                    ┌─────────────────────────────┼──────────────┐            │
│                    │              │              │              │            │
│                    ▼              ▼              ▼              ▼            │
│           ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐        │
│           │ DynamoDB  │   │   SNS    │   │   SQS    │   │   S3     │        │
│           │ 3 Tables  │   │  Topics  │   │  Queue   │   │ Media    │        │
│           │ GSI+TTL   │   │  Alerts  │   │  + DLQ   │   │ Reports  │        │
│           └──────────┘   └──────────┘   └──────────┘   └──────────┘        │
│                                                                              │
│  ┌──────────────────────────────────┐                                       │
│  │  VPC — 2 AZ, Public + Private   │                                       │
│  │  Subnets, IGW, Route Tables     │                                       │
│  └──────────────────────────────────┘                                       │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Event-Driven Data Flow

```
Citizen / First Responder / IoT Sensor
              │
              ▼
     ┌─────────────────┐
     │   API Gateway    │──── Rate limited: 50 req/sec
     │   (REST)         │     X-Ray tracing enabled
     └────────┬────────┘
              │
     ┌────────┼────────────────────┐
     │        │                    │
     ▼        ▼                    ▼
┌─────────┐ ┌───────────┐ ┌──────────────┐
│  POST   │ │   GET     │ │    POST      │
│/incident│ │/incidents │ │ /resources/  │
│         │ │ ?severity │ │   allocate   │
└────┬────┘ └───────────┘ └──────────────┘
     │
     ▼
┌──────────┐     ┌──────────┐
│ DynamoDB │     │   SNS    │ ─── Email / SMS / Push
│ (Store)  │     │ (Alert)  │     to first responders
└──────────┘     └────┬─────┘
                      │
                      ▼
                ┌──────────┐     ┌──────────┐
                │   SQS    │────▶│  Lambda   │ ─── Async processing
                │  (Queue) │     │ (Worker)  │     De-duplication
                └──────────┘     └──────────┘     Enrichment
                      │
                ┌─────▼──────┐
                │   DLQ      │ ─── Failed messages
                │ (14 days)  │     for investigation
                └────────────┘
```

---

## 📁 Repository Structure

```
aws-disaster-response-cloudformation/
├── 📄 README.md
├── infra/
│   ├── main.yaml                           # Root stack — orchestrates nested stacks
│   ├── modules/
│   │   ├── networking.yaml                 # VPC, subnets, IGW, route tables, SGs
│   │   ├── dynamodb.yaml                   # 3 DynamoDB tables with GSIs
│   │   ├── messaging.yaml                  # SNS topics + SQS queues + DLQ
│   │   ├── lambda.yaml                     # 3 Lambda functions + IAM role
│   │   ├── apigateway.yaml                 # REST API + methods + stage
│   │   └── s3.yaml                         # Media + reports buckets
│   └── parameters/
│       ├── dev.json                        # Dev environment params
│       ├── staging.json                    # Staging environment params
│       └── prod.json                       # Production environment params
└── pipelines/
    ├── ci-build.yaml                       # CI: lint, validate, package, upload to Azure Storage
    └── cd-release.yaml                     # CD: download from Azure Storage, deploy to AWS
```

---

## 🔧 Infrastructure Components

| Resource | Purpose | DEV | PROD |
|----------|---------|-----|------|
| **API Gateway** | REST API endpoint | Regional, throttled | Regional, X-Ray, throttled |
| **Lambda** | 3 functions (Python 3.12) | 256 MB | 512 MB |
| **DynamoDB** | 3 tables + GSIs | Pay-per-request | Provisioned, PITR enabled |
| **SNS** | 2 topics (alerts, coordination) | KMS encrypted | KMS encrypted |
| **SQS** | Queue + DLQ, long polling | 1-day retention | 1-day + 14-day DLQ |
| **S3** | Media + reports buckets | No versioning | Versioned, Glacier lifecycle |
| **VPC** | 2 AZ, public + private subnets | Standard | Standard |

### DynamoDB Data Model

| Table | Partition Key | GSIs | Purpose |
|-------|-------------|------|---------|
| `incidents` | `incidentId` | severity-time, region-time | All disaster incidents |
| `resources` | `resourceId` | type-status | Emergency resources |
| `volunteers` | `volunteerId` | region-skill | Volunteer registry |

### Lambda Functions

| Function | Trigger | Purpose |
|----------|---------|---------|
| `report-incident` | API Gateway POST + SQS | Report new disaster incident |
| `get-incidents` | API Gateway GET | Query incidents by severity/region |
| `allocate-resource` | API Gateway POST | Assign resources to incidents |

---

## 🚀 CI/CD Pipeline

### Key Design: Built on Azure DevOps, Deployed to AWS

```
Azure DevOps (TFS)                         Azure Storage           AWS
┌──────────────┐     ┌──────────────┐     ┌──────────┐     ┌──────────────┐
│  CI Pipeline │────▶│  Validate    │────▶│  Upload  │────▶│  CF Deploy   │
│  (YAML)      │     │  cfn-lint    │     │ Artifact │     │  (Nested     │
│              │     │  aws validate│     │ Versioned│     │   Stacks)    │
└──────────────┘     └──────────────┘     └──────────┘     └──────────────┘
                                                                  │
                                                     DEV → STAGING → PROD
```

### CI Pipeline — `ci-build.yaml`
1. **cfn-lint** — Lint all YAML templates
2. **aws cloudformation validate** — Validate against AWS API
3. **aws cloudformation package** — Resolve nested stacks, upload to S3
4. **Upload artifact** — Versioned artifact to Azure Storage Account

### CD Pipeline — `cd-release.yaml`
1. **Download** versioned artifact from Azure Storage
2. **Deploy to DEV** → auto
3. **Deploy to STAGING** → auto
4. **Deploy to PROD** → manual approval, change set review

---

## 🛠️ Getting Started

### Prerequisites

- AWS CLI ≥ 2.0 configured with credentials
- Azure CLI ≥ 2.50 (for artifact storage)
- Python 3.12 (for cfn-lint)
- Azure DevOps organization

### Local Deployment

```bash
# Validate template
aws cloudformation validate-template --template-body file://infra/main.yaml

# Package nested stacks
aws cloudformation package \
  --template-file infra/main.yaml \
  --s3-bucket your-cf-templates-bucket \
  --output-template-file packaged.yaml

# Deploy to dev
aws cloudformation deploy \
  --template-file packaged.yaml \
  --stack-name disaster-response-dev \
  --parameter-overrides file://infra/parameters/dev.json \
  --capabilities CAPABILITY_NAMED_IAM

# Delete stack
aws cloudformation delete-stack --stack-name disaster-response-dev
```

---

## 🔐 Security

- **IAM least-privilege** — Lambda role has minimal DynamoDB, SNS, SQS permissions
- **KMS encryption** — SNS topics and SQS queues encrypted at rest
- **S3 block public access** — All buckets fully private
- **Server-side encryption** — AES-256 on S3. SSE on DynamoDB
- **API Gateway throttling** — 50 req/sec burst: 100
- **VPC isolation** — Private subnets for internal workloads
- **DLQ** — Failed messages preserved for 14 days for investigation
- **X-Ray tracing** — End-to-end request tracing in API Gateway

---

## 📊 API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/incidents` | Report a new disaster incident |
| `GET` | `/incidents?severity=critical` | Get incidents by severity |
| `GET` | `/incidents?region=northeast` | Get incidents by region |
| `POST` | `/resources/allocate` | Allocate resource to incident |

---

## 📝 License

MIT License — see [LICENSE](LICENSE) for details.
