# Cost Estimates

**Project:** Aura Promptly  
**Version:** 1.1  
**Date:** 2026-05-14  
**Status:** Baselined (updated post-review)

---

## Change Log

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-05-14 | Initial baseline |
| 1.1 | 2026-05-14 | Section 2.3 updated: Aurora pgvector replaces OpenSearch Serverless (~90% cost reduction); Section 3 scenarios recalculated; Section 4.1 toggle variables updated; Section 4.3 recreate runbook updated with KB sync poll-and-wait |

> All prices are approximate, based on AWS us-east-1 pricing as of May 2026. Actual costs depend on usage volume. Use Infracost (`infracost breakdown --path ./infra`) for current exact estimates.

---

## 1. Cost Philosophy

The PoC infrastructure is designed to be **destroyed when not in use** and **recreated in under 30 minutes**. The cost strategy is:

- Pay only when actively developing or demoing
- Aurora Serverless scales to 0 ACUs when idle — the KB vector store costs nothing between sessions
- Enable/disable optional components (WAF, Redis, Grafana export) via Terraform variables
- The two highest-cost items in v1.0 (OpenSearch Serverless ~$172/month, NAT Gateway ~$32/month) are now either eliminated or toggleable

---

## 2. Component Cost Breakdown

### 2.1 Hub Account (AWS)

| Component | Service | Pricing Model | Always-on monthly cost | Active (8h/day, 20 days) |
|---|---|---|---|---|
| LiteLLM Proxy | ECS Fargate (0.25 vCPU, 0.5 GB) | Per task-hour | ~$3/month | ~$3.50/month |
| LiteLLM DB | Aurora Serverless (PostgreSQL) | Per ACU-hour (min 0.5 ACU, auto-pauses) | ~$4.50/month | ~$8/month |
| Response Cache | ElastiCache Redis (cache.t4g.micro) | Per hour | ~$12/month | ~$12/month |
| API Gateway | REST API | Per million requests | ~$0 (PoC volume) | <$1/month |
| WAF | AWS WAF WebACL | $5/WebACL + $1/million requests | $5/month | $5/month |
| Cognito | User Pool | Free up to 50k MAUs | $0 | $0 |
| Secrets Manager | Secrets | $0.40/secret/month | ~$2/month | ~$2/month |
| S3 | Storage (config, KB docs) | $0.023/GB | <$1/month | <$1/month |
| NAT Gateway | Per hour + data | $0.045/hour + $0.045/GB | ~$32/month (always on) | ~$32/month |
| VPC Endpoints | Per endpoint-hour | $0.01/endpoint/hour (7 endpoints) | ~$50/month | ~$50/month |
| CloudWatch Logs | Ingestion + storage | $0.50/GB ingested | ~$2/month | ~$3/month |
| OTel Collector | ECS Fargate (0.25 vCPU, 0.5 GB) | Per task-hour | ~$3/month | ~$3.50/month |

> **Note on NAT vs VPC Endpoints:** VPC endpoints cost ~$50/month (7 endpoints) but NAT Gateway costs ~$32/month. For small traffic volumes, NAT is cheaper. For high Bedrock traffic, VPC endpoints save on data transfer costs. The Terraform toggle `enable_vpc_endpoints` lets you switch.

**Hub base cost (always-on, no AI services running): ~$65–85/month**

To reduce hub cost during idle periods:
- Set ECS desired count to 0 (Fargate tasks stop immediately)
- Aurora auto-pauses after 5 minutes — effectively ~$0 when idle

**Hub cost when fully idle (ECS stopped, Aurora paused): ~$15–20/month** (NAT GW, VPC endpoints, Secrets Manager, S3)

### 2.2 AI Services (On-Demand, Usage-Based)

| Service | Unit Cost | PoC Estimate (100 requests/day) |
|---|---|---|
| Bedrock Claude Haiku 3 | $0.00025/1K input + $0.00125/1K output | ~$0.30/day (~500 input, 200 output tokens avg) |
| Bedrock Titan Embeddings v2 | $0.00002/1K tokens | ~$0.01/day (RAG indexing) |
| Bedrock Guardrails | $0.75/1K text units | ~$0.08/day |
| OpenAI GPT-4o-mini | $0.00015/1K input + $0.0006/1K output | ~$0.10/day (fallback only) |
| Vertex AI Gemini Flash | $0.000075/1K input + $0.0003/1K output | ~$0.05/day (if routed) |

**Total AI usage cost (100 requests/day active session): ~$0.54/day = ~$11/month**

### 2.3 Knowledge Base / RAG

| Component | Cost | Notes |
|---|---|---|
| Aurora Serverless (pgvector schema) | **~$0 when idle** (scale-to-zero) | Shared cluster with LiteLLM DB — no extra cluster cost |
| Aurora Serverless (active, 0.5 ACU) | ~$0.06/ACU-hour × 0.5 = ~$0.03/hour | Only bills when queries are running |
| Aurora Serverless (8h/day, 20 days) | ~$5–10/month | Realistic active development cost |
| S3 (document storage) | $0.023/GB | <$1/month for PoC docs |
| Bedrock KB ingestion | Included in embedding cost | ~$0.01 per sync |

> **Cost comparison vs v1.0:** OpenSearch Serverless cost $172/month (always-on) or $58/month (8h/day). Aurora pgvector costs ~$5–10/month active and **$0 when idle**. This is a ~90% reduction and eliminates the need to destroy the KB between sessions.

> **Production note:** At high QPS (>500 req/s), migrate to OpenSearch Serverless for better vector search performance. The Bedrock KB API is identical — only the backing store changes.

### 2.4 Spoke Accounts

| Spoke | Service | Monthly Cost |
|---|---|---|
| Azure (Container Apps) | Consumption plan (serverless) | ~$0–3/month (free tier covers PoC) |
| AWS (ECS Fargate or Lambda) | Per request / per task-hour | ~$2–5/month |
| GCP (Cloud Run) | Per request (serverless) | ~$0–2/month (free tier covers PoC) |

**Total spokes: ~$5–10/month**

### 2.5 Tooling (Free Tiers)

| Tool | Tier | Cost |
|---|---|---|
| Terraform Cloud | Free (1 user, 500 resources) | $0 |
| Grafana Cloud | Free (10k metrics, 50GB logs) | $0 |
| GitHub Actions | Free (2,000 min/month private repos) | $0 |

---

## 3. Cost Scenarios

### Scenario A: Actively Developing (8 hours/day, 5 days/week)

All components running, moderate AI usage:

| Category | Monthly Cost |
|---|---|
| Hub infrastructure (ECS, Aurora, Redis, WAF, NAT) | ~$65 |
| Aurora pgvector KB (8h/day × 20 days, 0.5 ACU) | ~$8 |
| AI usage (100 req/day) | ~$11 |
| Spokes | ~$8 |
| **Total** | **~$92/month** |

> **v1.0 comparison:** Was ~$152/month. Saving: ~$60/month from eliminating OpenSearch Serverless.

### Scenario B: Demo Day Only (2 hours, once)

| Category | Cost |
|---|---|
| Hub (2 hours) | ~$0.50 |
| Aurora pgvector KB (2 hours, 0.5 ACU) | ~$0.06 |
| AI usage (200 requests during demo) | ~$0.11 |
| Spokes (2 hours) | ~$0.10 |
| **Total** | **~$0.77 per demo session** |

> **v1.0 comparison:** Was ~$1.67 per session. OpenSearch alone was $0.96 for 2 hours.

### Scenario C: Idle (hub running, KB not actively queried)

Aurora scales to 0 ACUs after 5 minutes of inactivity.

| Category | Monthly Cost |
|---|---|
| NAT Gateway (always-on) | ~$32 |
| ECS Fargate (LiteLLM, 1 task) | ~$3 |
| Aurora Serverless (paused, 0 ACU) | ~$0 |
| WAF | ~$5 |
| Secrets Manager, S3, CloudWatch | ~$5 |
| **Total** | **~$45/month** |

### Scenario D: Fully Destroyed (no active infrastructure)

**Cost: $0/day** (Terraform state in Terraform Cloud, code in GitHub — no AWS resources running)

---

## 4. Cost Control Mechanisms

### 4.1 Terraform Toggle Variables

```hcl
# infra/hub/variables.tf
variable "enable_waf"              { default = true  }  # saves $5/month if false
variable "enable_redis_cache"      { default = true  }  # saves $12/month if false
variable "enable_vpc_endpoints"    { default = false }  # saves $50/month if false (use NAT instead)
variable "enable_knowledge_base"   { default = false }  # drops Aurora pgvector schema when false
variable "enable_grafana_export"   { default = true  }  # saves ~$0 (free tier) but stops data egress
variable "enable_otel_collector"   { default = true  }  # saves ~$3/month if false
```

**Minimum viable PoC cost (WAF off, KB off, Redis off, VPC endpoints off, Grafana off):**  
~$20–30/month when actively running, ~$5/month idle.

### 4.2 Destroy Runbook

```bash
# Step 1: Destroy spokes first (to remove API keys and IAM roles referencing hub)
terraform -chdir=infra/gcp-spoke destroy -auto-approve
terraform -chdir=infra/azure-spoke destroy -auto-approve
terraform -chdir=infra/aws-spoke destroy -auto-approve

# Step 2: Destroy hub
terraform -chdir=infra/hub destroy -auto-approve

# Verification: no resources should remain in AWS account
aws resourcegroupstaggingapi get-resources \
  --tag-filters Key=project,Values=aura-promptly \
  --query 'ResourceTagMappingList[*].ResourceARN'
```

### 4.3 Recreate Runbook

```bash
# Step 1: Deploy hub
terraform -chdir=infra/hub apply -auto-approve

# Step 2: Upload KB documents to S3
aws s3 sync ./data/kb-documents/ s3://$(terraform -chdir=infra/hub output -raw kb_bucket_name)/

# Step 3: Trigger Knowledge Base sync
JOB_ID=$(aws bedrock-agent start-ingestion-job \
  --knowledge-base-id $(terraform -chdir=infra/hub output -raw kb_id) \
  --data-source-id $(terraform -chdir=infra/hub output -raw kb_data_source_id) \
  --query 'ingestionJob.ingestionJobId' --output text)

# Step 4: Poll until sync completes (MUST complete before spokes start serving RAG queries)
echo "Waiting for KB sync to complete (job: $JOB_ID)..."
while true; do
  STATUS=$(aws bedrock-agent get-ingestion-job \
    --knowledge-base-id $(terraform -chdir=infra/hub output -raw kb_id) \
    --data-source-id $(terraform -chdir=infra/hub output -raw kb_data_source_id) \
    --ingestion-job-id $JOB_ID \
    --query 'ingestionJob.status' --output text)
  echo "  Status: $STATUS"
  [ "$STATUS" = "COMPLETE" ] && echo "KB sync complete." && break
  [ "$STATUS" = "FAILED" ]   && echo "KB sync FAILED — check CloudWatch logs." && exit 1
  sleep 15
done

# Step 5: Deploy spokes (safe to start now — KB is ready)
terraform -chdir=infra/aws-spoke apply -auto-approve
terraform -chdir=infra/azure-spoke apply -auto-approve
terraform -chdir=infra/gcp-spoke apply -auto-approve
```

> **Why the poll step matters:** Without it, the Azure RAG chatbot returns empty results immediately after a fresh deploy. The sync typically takes 1–5 minutes for small document sets — well within the NFR-02 <30 minute recreate target.

### 4.4 Infracost Integration

Every `terraform plan` in CI/CD will include an Infracost cost estimate:

```yaml
# .github/workflows/terraform-pr.yml (excerpt)
- name: Run Infracost
  uses: infracost/actions/setup@v3
  with:
    api-key: ${{ secrets.INFRACOST_API_KEY }}

- name: Generate Infracost diff
  run: infracost diff --path infra/hub --compare-to infracost-base.json

- name: Post cost estimate to PR
  uses: infracost/actions/comment@v3
  with:
    path: /tmp/infracost.json
    behavior: update
```

This posts a cost delta comment on every PR, showing the cost impact of infrastructure changes.

---

## 5. AWS Billing Alerts

The following CloudWatch billing alarms will be created by Terraform:

| Alarm | Threshold | Action |
|---|---|---|
| Monthly estimated charges | $50 | Email notification |
| Monthly estimated charges | $100 | Email notification |
| Monthly estimated charges | $200 | Email notification (warning) |

```hcl
# Billing alarms require CloudWatch in us-east-1 regardless of hub region
resource "aws_cloudwatch_metric_alarm" "billing_50" {
  alarm_name          = "aura-promptly-billing-50usd"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "EstimatedCharges"
  namespace           = "AWS/Billing"
  period              = 86400
  statistic           = "Maximum"
  threshold           = 50
  alarm_actions       = [aws_sns_topic.billing_alerts.arn]
}
```

---

## 6. Cost Comparison: PoC vs Production

| Aspect | PoC | Production (indicative) |
|---|---|---|
| LiteLLM compute | 1 Fargate task (0.25 vCPU) | 3–10 tasks (auto-scaling) |
| Aurora | Serverless, scale-to-zero, shared cluster | Provisioned (2 ACU min), Multi-AZ, dedicated cluster |
| Vector store | Aurora pgvector (shared cluster) | OpenSearch Serverless (4+ OCU) or dedicated pgvector |
| WAF | Basic rules | Advanced rules + Bot Control |
| Bedrock model | Haiku 3 | Sonnet 3.5 or higher |
| Monthly cost | $20–$110 (toggleable) | $2,000–$10,000+ |

The PoC is intentionally 20–100x cheaper than production to enable learning without financial risk.
