# AI-Powered Root Cause Analysis (RCA) System
> Automated failure detection, AI-powered root cause analysis, and engineering notification for AWS Lambda applications.

---

## Problem Statement

Investigating production failures often requires engineers to manually analyze logs, metrics, and source code, increasing the time required to identify root causes and resolve incidents, leading to prolonged downtime and delayed response.

---

## Architecture Diagram

![AI RCA System Architecture](AI_RCA_Architecture.svg)

---

## Assumptions

- TypeScript Lambda running on Node.js 22.x
- Structured JSON logging enabled on application Lambda
- Source maps enabled with `NODE_OPTIONS=--enable-source-maps`
- CloudWatch Logs is the primary source of error and stack trace information.
- SNS acts as intermediary between CloudWatch Alarm and RCA Lambda
- Amazon SES configured with verified sender domain
- Only stack trace referenced source files are fetched from GitHub (maximum 5 files with 50–100 lines of context per file)
- RCA Lambda has IAM permissions for CloudWatch Logs, CloudWatch Metrics, Bedrock, SES and Secrets Manager

---

## Component Responsibilities

| Component | Responsibility |
|---|---|
| AI App Lambda | Business logic, emits structured error logs with stack traces |
| CloudWatch Logs | Stores error logs, stack traces and error payload |
| CloudWatch Metrics | Tracks Errors, Duration, Invocations, Throttles, Concurrent Executions |
| CloudWatch Alarm | Triggers on Errors >= 1 per 1-minute period |
| SNS Topic | Bridges alarm state change to RCA Lambda invocation |
| RCA Orchestrator Lambda | Queries logs and metrics, parses stack trace, fetches GitHub files, builds prompt, calls Bedrock, sends report |
| GitHub Repository | Source of truth for application code — returns referenced files only |
| Amazon Bedrock | Claude Sonnet — generates root cause analysis with confidence score |
| Amazon SES | Delivers RCA report to engineering team |

---

## Trigger Mechanism

Event-driven via CloudWatch Alarm → SNS → Lambda. Chosen over polling for instant detection, zero idle compute cost, and no additional scheduler needed.

---

## Data Flow

```
1. FAILURE DETECTED
   AI App Lambda throws error
   → Structured error log (message + stack trace) 
     written to CloudWatch Logs
   → Lambda Errors metric incremented in 
     CloudWatch Metrics

2. ALARM TRIGGERED
   CloudWatch Alarm evaluates Errors >= 1
   → State transitions OK → ALARM
   → Publishes event to SNS Topic

3. RCA LAMBDA INVOKED
   SNS invokes RCA Orchestrator Lambda
   → Alarm context + timestamp passed as payload

4. DIAGNOSTIC DATA COLLECTION
   RCA Lambda runs parallel data fetch:
   → CloudWatch Logs Insights query (5 min window)
     — Retrieves ERROR logs + full stack trace
   → GetMetricData call
     — Retrieves Errors, Duration, Invocations,
       Throttles, Concurrent Executions

5. SOURCE CODE CONTEXT
   RCA Lambda parses stack trace from error log
   → Extracts file names + line numbers
   → Fetches referenced files from GitHub via 
     token-based API (max 5 files, 50-100 lines)

6. AI ANALYSIS
   Builds structured prompt:
   → Logs + Stack trace + Metrics + Code context
   → Sends to Amazon Bedrock (Claude Sonnet)
   → Receives: Root Cause, Evidence, Impact, 
     Recommended Fix, Confidence Score

7. NOTIFICATION
   RCA Lambda sends formatted report via SES
   → Delivered to engineering@company.com
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Runtime | Node.js 22.x + TypeScript |
| Compute | AWS Lambda |
| Observability | CloudWatch Logs + Metrics |
| Alerting | CloudWatch Alarm + SNS |
| Source Context | GitHub REST API |
| AI Model | Amazon Bedrock — Claude Sonnet |
| Notification | Amazon SES |

---

## Security

- Least privilege IAM per Lambda
- Only stack trace referenced source files sent to AI model — no full repository access
- GitHub API access via token-based auth — token stored and retrieved from AWS Secrets Manager
- Structured logging only — no PII in logs

---
## Limitations

- Root cause accuracy depends on the quality of application logs.
- Source code context is limited to stack-trace-referenced files.
- The system provides recommendations and confidence scores but does not perform automatic remediation.
- External dependency failures may require additional tracing tools such as AWS X-Ray.
  
---
## Future Enhancements

| Enhancement | Reason |
|---|---|
| X-Ray tracing integration | Request-level trace for deeper dependency analysis |
| Slack / GChat notification support | Multi-channel alerting beyond email |
| Auto Jira ticket creation | Reduces manual overhead for incident tracking |
| Historical incident knowledge base | Pattern detection across repeated failures |
| SQS + DLQ before SES | Guaranteed email delivery under high alarm volume |
