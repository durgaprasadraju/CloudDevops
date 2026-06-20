# MASTER AWS Lambda Guide

> **Complete learning guide for AWS Developer Associate · Solutions Architect Associate · DevOps Engineer · Backend Engineer interviews**

Synthesized from all modules in this repository. For deep dives, follow the linked module guides.

| Module | Detailed Guide |
|--------|----------------|
| Fundamentals | [1.Fundamentals/README.md](./1.Fundamentals/README.md) |
| Advanced | [2.Advanced/README.md](./2.Advanced/README.md) |
| Integrations | [3.Integrations/README.md](./3.Integrations/README.md) |
| Security & Monitoring | [4.Security-Monitoring/README.md](./4.Security-Monitoring/README.md) |
| Deployment | [5.Deployment/README.md](./5.Deployment/README.md) |

---

## How to Use This Guide

### Recommended Learning Path

```
Part 1 Fundamentals     →  What Lambda is, handler, memory, timeout
Part 2 Advanced         →  Versions, concurrency, cold starts
Part 3 Integrations     →  API GW, S3, SQS, streams, RDS
Part 4 Security         →  IAM, secrets, CloudWatch, X-Ray
Part 5 Deployment       →  SAM, CDK, Terraform, CI/CD
Cheat Sheet             →  Review before exam or interview
```

### Audience Focus Areas

| Role / Exam | Priority Topics | Sections |
|-------------|-----------------|----------|
| **AWS Developer Associate** | Handler, integrations, deployment, IAM, CloudWatch | Parts 1, 3, 4, 5 |
| **Solutions Architect Associate** | When to use Lambda, integrations, limits, cost, security | Parts 1, 2, 3, 4 |
| **DevOps Engineer** | Deployment, CI/CD, monitoring, alarms, concurrency | Parts 2, 4, 5 |
| **Backend Engineer Interview** | All parts + Interview Question Bank | All + Cheat Sheet |

---

# Part 1 — Fundamentals

## 1.1 What Is AWS Lambda?

**AWS Lambda** is a **serverless, event-driven compute service**. You upload code; AWS runs it when triggered — no servers to provision, patch, or scale.

> Lambda is a compute service that lets you run code without provisioning or managing servers. AWS handles server maintenance, capacity provisioning, automatic scaling, and logging. You pay only for compute time consumed.

### Key Characteristics

| Feature | Detail |
|---------|--------|
| Compute model | Event-driven functions |
| Scaling | Automatic — scales to zero |
| Billing | Per request + duration (GB-seconds) |
| Free tier | 1M requests + 400K GB-seconds/month |
| Languages | Python, Node.js, Java, Go, .NET, Ruby, etc. |

### Serverless ≠ No Servers

**Serverless** means **you don't manage servers**. AWS does. Lambda is **FaaS** (Function as a Service) — you manage only the **function code**.

```
IaaS (EC2)     → You manage OS + App
PaaS           → You manage App
FaaS (Lambda)  → You manage Code only
```

### Evolution

```
Physical Server → EC2 → Containers → Lambda (Serverless)
Pay 24/7          Pay/hour  Manage images   Pay/millisecond, scale to zero
```

---

## 1.2 Benefits & Limitations

### Benefits

1. **No server management** — no SSH, patching, or capacity planning
2. **Automatic scaling** — 1 request or 10,000 requests handled automatically
3. **Pay per use** — no cost when idle
4. **High availability** — multi-AZ by default
5. **Fast deployment** — minutes, not days
6. **200+ AWS integrations** — native event sources

### When NOT to Use Lambda

| Scenario | Better Alternative |
|----------|-------------------|
| Full OS control | EC2, ECS |
| Workloads > 15 minutes | EC2, Step Functions |
| Steady 24/7 high traffic | EC2 Reserved Instances |
| GPU workloads | EC2 with GPU |
| WebSocket long connections | API Gateway WebSocket + DynamoDB |
| Large packages (> 250 MB zip) | Container image (10 GB) or ECS |

### Service Limits (Must Know)

| Limit | Value |
|-------|-------|
| Max timeout | **900 sec (15 min)** |
| Memory | **128 MB – 10,240 MB** |
| Env variables total | **4 KB** |
| Deployment package (zip) | 50 MB zipped / 250 MB unzipped |
| Container image | Up to **10 GB** |
| Layers per function | **5** |
| Concurrent executions (default) | **1,000/region** |
| `/tmp` storage | 512 MB – 10,240 MB |

---

## 1.3 Architecture

```
Event Source  →  Lambda (Handler + Runtime)  →  Downstream Services
(S3, API GW)     (Sandbox isolation)            (DynamoDB, S3, SNS)
```

### Components

| Component | Role |
|-----------|------|
| **Event source** | Triggers Lambda (S3, API GW, SQS…) |
| **Handler** | Entry point function |
| **Runtime** | Language environment (Python 3.12, etc.) |
| **Execution role** | IAM role — outbound permissions |
| **Environment variables** | Non-sensitive config |
| **Layers** | Shared dependencies |

### Runtime Lifecycle

```
INIT (cold start) → INVOKE (handler runs) → SHUTDOWN (after idle ~10–15 min)
```

---

## 1.4 Handler, Runtime, Memory, Timeout

### Handler (Python)

```python
def lambda_handler(event, context):
    # event   = input from trigger
    # context = request_id, memory, remaining time
    return {"statusCode": 200, "body": "OK"}
```

**Format:** `filename.function_name` → `lambda_function.lambda_handler`

### Memory & CPU

- Memory: **128 MB – 10 GB** (1 MB increments)
- **More memory = more CPU** — sometimes faster AND cheaper
- Sweet spot: load-test at 128, 256, 512, 1024 MB

### Timeout

- Default: **3 sec** | Max: **900 sec (15 min)**
- Use `context.get_remaining_time_in_millis()` for graceful exit

### Environment Variables

- Max **4 KB** total
- Use for **non-sensitive config** (table names, URLs)
- **Never** store passwords — use Secrets Manager

---

## 1.5 Use Cases

| Use Case | Trigger |
|----------|---------|
| REST API backend | API Gateway |
| Image processing | S3 ObjectCreated |
| Scheduled jobs | EventBridge cron |
| Stream processing | Kinesis / DynamoDB Streams |
| Async workflows | SQS → Lambda |
| Notifications | SNS fan-out |

---

# Part 2 — Advanced Concepts

## 2.1 Versions & Aliases

| Concept | Detail |
|---------|--------|
| **$LATEST** | Mutable working copy — **never for prod** |
| **Published version** | Immutable snapshot (1, 2, 3…) |
| **Alias** | Stable pointer (`prod`, `staging`) to a version |

### Canary Deployment

```bash
# Route 10% traffic to new version 6, 90% stays on version 5
aws lambda update-alias --function-name my-fn --name prod \
  --function-version 5 \
  --routing-config AdditionalVersionWeights={"6"=0.1}
```

**Production rule:** API Gateway and triggers point to **alias**, not `$LATEST`.

---

## 2.2 Layers

- ZIP archive of libraries mounted at `/opt`
- Max **5 layers** per function
- Share dependencies across functions (pandas, internal utils)
- Python path: `python/lib/python3.12/site-packages/`

---

## 2.3 Concurrency

```
Concurrent execution = 1 Lambda instance handling 1 request at a time
Account default limit = 1,000 per region (soft limit, can increase)
```

| Type | Purpose |
|------|---------|
| **Concurrency (default)** | Auto-scale on demand |
| **Reserved concurrency** | Cap AND guarantee capacity per function |
| **Provisioned concurrency** | Pre-warm environments — eliminate cold starts |

| Setting | Reserved = 0 | Reserved = 100 | Provisioned = 20 |
|---------|-------------|----------------|------------------|
| Effect | Function disabled | Max 100, guaranteed 100 | 20 warm instances always ready |
| Cost | — | Standard invoke pricing | Hourly PC charge + invokes |

---

## 2.4 Cold Starts & Execution Environment

### Cold Start

Extra latency when Lambda creates a new execution environment (INIT phase).

| Runtime | Typical Cold Start |
|---------|-------------------|
| Python / Node.js | 100–400 ms |
| Go | 50–200 ms |
| Java | 1–3+ seconds |

### Mitigation

- Smaller deployment package
- Lambda Layers for heavy deps
- Lazy-load imports
- Increase memory (more CPU during init)
- **Provisioned Concurrency** for latency-critical APIs
- Avoid VPC unless required

### Execution Environment Reuse

| Persists (warm) | Does NOT persist |
|-----------------|------------------|
| Global variables | After idle timeout |
| `/tmp` contents | Across environments |
| DB connection pools | Indefinitely |

**VPC adds 1–10+ sec** to cold starts (ENI creation). Use **RDS Proxy** for connection pooling.

---

# Part 3 — Integrations

## 3.1 Integration Summary

| Service | Direction | Invocation | Batch? |
|---------|-----------|------------|--------|
| **API Gateway** | Trigger | **Sync** | No |
| **S3** | Trigger | Async | No |
| **SQS** | Trigger | Poll | Yes (up to 10) |
| **SNS** | Trigger | Async (fan-out) | No |
| **EventBridge** | Trigger | Async | No |
| **DynamoDB Streams** | Trigger | Poll | Yes |
| **Kinesis** | Trigger | Poll | Yes |
| **RDS** | Target | Lambda calls DB | N/A |

### Decision Guide

```
HTTP API?           → API Gateway + Lambda
File upload?        → S3 + Lambda
Buffer/decouple?    → SQS + Lambda
Notify many?        → SNS + Lambda
Event routing?      → EventBridge + Lambda
DB change react?    → DynamoDB Streams + Lambda
Real-time stream?   → Kinesis + Lambda
Relational data?    → Lambda + RDS Proxy + RDS (VPC)
```

---

## 3.2 Key Integration Patterns

### API Gateway (Sync)

```
Client → API Gateway → Lambda → DynamoDB → Response to client
```

Lambda returns: `{ statusCode, headers, body }`

### S3 (Async)

```
Upload → S3 → ObjectCreated event → Lambda → Process → S3 output
```
Retries: 2x on failure. Avoid writing to same trigger prefix (infinite loop).

### SQS (Poll via Event Source Mapping)

```
Producer → SQS Queue → Lambda polls batch → Process → DeleteMessage
```
Failed messages: visibility timeout retry → DLQ after maxReceiveCount.

### SNS (Fan-out)

```
Publisher → SNS Topic → Lambda A + Lambda B + SQS + Email (parallel)
```

### EventBridge (Event Bus + Schedule)

```
PutEvents → Rule (pattern match) → Lambda / SQS / Step Functions
cron(0 8 * * ? *) → Scheduled Lambda
```

### DynamoDB Streams (CDC)

```
DynamoDB write → Stream record → Lambda → OpenSearch / S3 / Analytics
```
Delivery: **at-least-once** — handler must be **idempotent**. Monitor **IteratorAge**.

### Kinesis (Real-time Stream)

```
Producers → Kinesis shards → Lambda (1 instance/shard default) → Process
```
Ordered per **partition key**. High throughput. Monitor **IteratorAge**.

### RDS (Lambda as Client)

```
API Gateway → Lambda (VPC) → RDS Proxy → Aurora/RDS
Credentials: Secrets Manager. Never hardcode passwords.
```

---

## 3.3 Error Handling by Integration

| Integration | On Failure |
|-------------|-----------|
| API Gateway | Error returned to client immediately |
| S3 | 2 retries → discard (configure DLQ) |
| SQS | Re-queue until maxReceiveCount → DLQ |
| SNS | 3 retries → discard |
| EventBridge | 24h retry with backoff → DLQ |
| DynamoDB Streams | Retry until data expires (24h) |
| Kinesis | Retry batch, bisect on error |

---

# Part 4 — Security & Monitoring

## 4.1 Security Model

```
INBOUND  → Resource Policy (who can invoke Lambda?)
OUTBOUND → Execution Role (what can Lambda access?)
SECRETS  → Secrets Manager / Parameter Store SecureString
ENCRYPT  → KMS (env vars, logs, secrets)
```

### IAM Execution Role

- Lambda **assumes** this role at runtime
- Trust policy: `lambda.amazonaws.com`
- Grant **least privilege** — specific ARNs, not `*`
- Managed policies: `AWSLambdaBasicExecutionRole`, `AWSLambdaVPCAccessExecutionRole`

### Resource Policy

- Attached **to the function**
- Controls inbound invoke (S3, SNS, API GW, cross-account)
- Use `AWS:SourceArn` and `AWS:SourceAccount` conditions

### KMS

- Encrypt env vars: set `KMSKeyArn` on function
- Encrypt log groups: `aws logs associate-kms-key`
- Lambda role needs `kms:Decrypt`

### Secrets Manager vs Parameter Store

| | Secrets Manager | Parameter Store |
|--|----------------|-----------------|
| Best for | Rotating credentials | Hierarchical config |
| Rotation | Automatic | Manual |
| Cost | Per secret/month | Free (Standard) |
| Example | DB password | `/app/prod/table-name` |

**Rule:** Never store passwords in environment variables.

---

## 4.2 Monitoring

### CloudWatch Logs

- Auto-created: `/aws/lambda/<function-name>`
- Requires `AWSLambdaBasicExecutionRole`
- Use **structured JSON** logging with `request_id`
- Set **retention** (14–90 days) — never infinite

### CloudWatch Metrics (Built-in)

| Metric | Alarm? |
|--------|--------|
| **Errors** | ✅ Critical |
| **Throttles** | ✅ Critical |
| **Duration** (p99) | ✅ SLA |
| **ConcurrentExecutions** | ⚠️ Capacity |
| **IteratorAge** | ✅ Stream lag |
| **DeadLetterErrors** | ✅ Critical |

### X-Ray

- **Active tracing:** `TracingConfig Mode=Active`
- Shows latency breakdown across services
- Custom subsegments for business logic bottlenecks
- Detect cold starts via INIT segment

### Alarms → SNS → PagerDuty/Slack

```
Errors > 0 for 3 min     → Critical → Page on-call
Throttles > 0            → Critical
Duration p99 > 3000ms    → Warning
IteratorAge > 60000ms    → Warning (stream lag)
```

**Production:** Alarm on **alias** (`payment-api:prod`), not `$LATEST`. Tie alarms to **CodeDeploy rollback** during canary.

---

# Part 5 — Deployment

## 5.1 Deployment Methods

| Method | Type | Best For |
|--------|------|----------|
| **Console** | Manual | Learning only |
| **CLI** | Script | CI scripts, quick updates |
| **SAM** | AWS IaC | Serverless-first apps |
| **CDK** | Code → CFN | Python/TS teams |
| **Terraform** | HCL IaC | Multi-cloud shops |
| **CloudFormation** | AWS IaC | Full control |
| **Serverless Framework** | Third-party | Fast prototyping |

```
SAM / CDK / Serverless  →  CloudFormation  →  AWS Resources
Terraform               →  AWS API directly
CLI / Console           →  Lambda API directly
```

---

## 5.2 Production Deployment Flow

```
1. Write code + tests
2. Package (pip install -t, zip, or sam build)
3. Deploy to dev (SAM/CDK/TF)
4. Integration tests
5. Publish version
6. Update alias (canary 5% → 100%)
7. Smoke test + monitor alarms
8. Rollback alias on error
```

### CI/CD Essentials

- **OIDC / IAM roles** — no static AWS keys in GitHub
- Separate **environments** (dev/staging/prod)
- **Manual approval** for prod
- Deploy to **alias**, not `$LATEST`
- **Canary** with CloudWatch alarm rollback

### Quick Deploy Commands

```bash
# CLI
aws lambda update-function-code --function-name my-fn --zip-file fileb://function.zip

# SAM
sam build && sam deploy --config-env prod

# CDK
cdk deploy MyStack

# Terraform
terraform apply

# Serverless
serverless deploy --stage prod
```

---

# Certification Topic Mapping

## AWS Developer Associate (DVA-C02)

| Domain | Lambda Topics in This Guide |
|--------|----------------------------|
| Development with AWS Services | Handler, boto3, integrations (API GW, S3, SQS, DynamoDB) |
| Security | IAM execution role, resource policy, Secrets Manager |
| Deployment | SAM, CLI, CI/CD, versions/aliases |
| Troubleshooting | CloudWatch Logs, X-Ray, error handling |

## AWS Solutions Architect Associate (SAA-C03)

| Domain | Lambda Topics in This Guide |
|--------|----------------------------|
| Design Resilient Architectures | SQS buffer, DLQ, multi-AZ, async vs sync |
| Design High-Performing | Memory/CPU, Provisioned Concurrency, cold starts |
| Design Cost-Optimized | Pay-per-use vs EC2, arm64, right-sizing memory |
| Design Secure | IAM least privilege, KMS, VPC, Secrets Manager |

## DevOps / Backend Interview Themes

| Theme | Key Points |
|-------|-----------|
| Serverless vs containers | Lambda for event-driven; ECS/EKS for long-running |
| Idempotency | Required for SQS, streams (at-least-once delivery) |
| Observability | Logs + metrics + traces + alarms |
| Safe deploys | Versions, aliases, canary, rollback |
| Scale limits | 1,000 concurrent default, reserved concurrency |

---

# Master Interview Question Bank

Consolidated from all modules. Answers are concise for interview prep.

## Core Concepts

**What is AWS Lambda?**
> Serverless, event-driven compute. Run code without managing servers. Pay per request and duration.

**Sync vs async invocation?**
> Sync: caller waits (API Gateway). Async: fire-and-forget with retries (S3, SNS, EventBridge).

**Max timeout?** 15 minutes (900 sec). Use Step Functions for longer workflows.

**How is Lambda billed?**
> Requests ($0.20/1M) + GB-seconds + Provisioned Concurrency if configured. Free tier: 1M requests/month.

**Lambda vs EC2?**
> Lambda: event-driven, variable traffic, short tasks, no ops. EC2: long-running, OS control, steady load, GPU.

## Advanced

**$LATEST vs published version?**
> $LATEST is mutable. Published versions are immutable. Prod uses versions via aliases.

**Reserved vs Provisioned concurrency?**
> Reserved: cap/guarantee capacity. Provisioned: pre-warm environments to eliminate cold starts.

**What causes cold starts?**
> New execution environment — INIT phase loads runtime, code, layers. Mitigate with smaller packages, PC, faster runtimes.

**What persists between invocations?**
> Global variables and `/tmp` in the same warm environment — not across cold starts.

## Integrations

**Event source mapping?**
> Lambda polls SQS, Kinesis, DynamoDB Streams and invokes with batches.

**SQS vs SNS vs EventBridge?**
> SQS: queue/buffer. SNS: fan-out pub/sub. EventBridge: event bus with routing rules and scheduling.

**Why RDS Proxy with Lambda?**
> Lambda scales to many instances — each could open a DB connection. Proxy pools connections.

**DynamoDB Streams exactly-once?**
> No — at-least-once. Handlers must be idempotent.

## Security & Monitoring

**Execution role vs resource policy?**
> Role: outbound (Lambda → AWS). Resource policy: inbound (who invokes Lambda).

**Where to store DB passwords?**
> Secrets Manager with rotation — not environment variables.

**Metrics to alarm on?**
> Errors, Throttles, Duration p99, IteratorAge, DeadLetterErrors.

## Deployment

**SAM vs CDK vs Terraform?**
> SAM: serverless YAML → CFN. CDK: code → CFN. Terraform: HCL, multi-cloud.

**CI/CD without AWS access keys?**
> GitHub Actions OIDC → assume IAM role with short-lived credentials.

**Canary deployment?**
> Alias weighted routing (10% new version) + CloudWatch alarms + CodeDeploy auto-rollback.

## System Design

**Design serverless order processing:**
> API Gateway → Lambda (validate) → DynamoDB → Stream → analytics Lambda + EventBridge → SNS (notify) + SQS → fulfillment Lambda.

**10,000 S3 uploads — how does Lambda scale?**
> One event per object. Lambda scales concurrently up to account limit (1,000 default). Use SQS if you need buffering.

---

# One-Page Cheat Sheet

> Print this page. Review before exams and interviews.

---

### LAMBDA AT A GLANCE

| Item | Value |
|------|-------|
| Model | Serverless, event-driven, FaaS |
| Max timeout | **900 sec (15 min)** |
| Memory | **128 MB – 10,240 MB** (more memory = more CPU) |
| Env vars limit | **4 KB** total |
| Package (zip) | 50 MB zipped / 250 MB unzipped |
| Container image | Up to **10 GB** |
| Layers | Max **5** per function |
| Concurrency default | **1,000/region** (soft limit) |
| Free tier | 1M req + 400K GB-sec/month |
| Billing | Requests + GB-seconds + PC hourly |

---

### HANDLER (Python)

```python
def lambda_handler(event, context):
    # event = trigger payload | context = request_id, memory, time left
    return {"statusCode": 200, "body": json.dumps({})}  # API GW format
```
**Handler string:** `file.function` → `lambda_function.lambda_handler`

---

### INVOCATION TYPES

| Trigger | Type | Retry |
|---------|------|-------|
| API Gateway | **Sync** | Client handles |
| S3, SNS, EventBridge | **Async** | 2–3 retries → DLQ |
| SQS, Kinesis, DDB Streams | **Poll (batch)** | Queue/stream retry → DLQ |

---

### INTEGRATIONS — PICK ONE

| Need | Use |
|------|-----|
| REST API | API Gateway |
| File upload event | S3 |
| Queue / buffer | SQS |
| Fan-out notify | SNS |
| Event routing / cron | EventBridge |
| DB change capture | DynamoDB Streams |
| Real-time stream | Kinesis |
| SQL database | Lambda (VPC) + RDS Proxy |

---

### ADVANCED CONTROLS

| Feature | Purpose |
|---------|---------|
| **Version** | Immutable snapshot for rollback |
| **Alias** | Stable prod pointer; canary weights |
| **Layer** | Shared deps at `/opt` (max 5) |
| **Reserved concurrency** | Cap/guarantee per function (=0 disables) |
| **Provisioned concurrency** | Pre-warmed — no cold start |
| **DLQ** | Failed async events destination |

---

### SECURITY CHECKLIST

```
✓ Least-privilege IAM execution role (no admin)
✓ Resource policy with SourceArn conditions
✓ Secrets in Secrets Manager (not env vars)
✓ KMS for env vars + logs (compliance)
✓ VPC only when accessing private resources
✓ No secrets in Git or logs
```

---

### MONITORING CHECKLIST

```
✓ CloudWatch Logs — JSON + request_id + retention set
✓ Alarm: Errors, Throttles, Duration p99
✓ Alarm: IteratorAge (streams), DeadLetterErrors
✓ X-Ray Active on user-facing functions
✓ Dashboard: invocations, errors, duration, concurrency
✓ Alarm on alias (prod), not $LATEST
```

---

### DEPLOYMENT QUICK COMMANDS

```bash
aws lambda update-function-code --function-name FN --zip-file fileb://fn.zip
aws lambda publish-version --function-name FN
aws lambda update-alias --function-name FN --name prod --function-version N
sam build && sam deploy --config-env prod
cdk deploy | terraform apply | serverless deploy --stage prod
```

---

### COLD START FIXES

`Smaller package → Layers → Lazy imports → More memory → arm64 → Provisioned Concurrency → Avoid VPC`

---

### WHEN NOT TO USE LAMBDA

`>15 min runtime · 24/7 steady load · GPU · full OS control · WebSocket long-lived · huge monolith`

---

### COST FORMULA

```
Cost = (Requests × $0.20/1M) + (GB-seconds × $0.0000166667)
GB-seconds = (Memory GB) × (Duration seconds)
```

---

### EXAM TRIGGERS → SERVICES

| Scenario | Answer |
|----------|--------|
| HTTP request-response | API Gateway + Lambda (sync) |
| Decouple microservices | SQS + Lambda |
| One event, many actions | SNS fan-out |
| Schedule daily job | EventBridge cron rule |
| React to DynamoDB insert | DynamoDB Streams |
| High-volume ordered stream | Kinesis |
| Too many DB connections | RDS Proxy |
| Eliminate cold start latency | Provisioned Concurrency |
| Safe prod deployment | Version + Alias + Canary |
| At-least-once stream processing | Idempotent handler + IteratorAge alarm |

---

### REPO MODULE INDEX

| # | Module | Path |
|---|--------|------|
| 1 | Fundamentals | [1.Fundamentals/README.md](./1.Fundamentals/README.md) |
| 2 | Advanced | [2.Advanced/README.md](./2.Advanced/README.md) |
| 3 | Integrations | [3.Integrations/README.md](./3.Integrations/README.md) |
| 4 | Security & Monitoring | [4.Security-Monitoring/README.md](./4.Security-Monitoring/README.md) |
| 5 | Deployment | [5.Deployment/README.md](./5.Deployment/README.md) |

---

*AWS Lambda, Python (Boto3) & Serverless — Beginner to Advanced · Master Guide*
