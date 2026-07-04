# Aura Promptly — Centralised GenAI Governance Platform

> **PoC Goals:** Hands-on learning · LinkedIn article · YouTube video

A production-grade proof of concept demonstrating how to centralise AI governance, guardrails, observability and cost control across a multi-cloud enterprise, with application teams in AWS, Azure and GCP consuming AI via a single, governed hub.

---

## What This PoC Demonstrates

| Concern | Solution |
|---|---|
| Central AI gateway | LiteLLM Proxy (ECS Fargate) |
| Model access | Amazon Bedrock (Claude Haiku 3, Titan Embeddings) |
| External model plug-in | OpenAI GPT-4o-mini, Vertex AI Gemini Flash |
| Agentic workflows | Amazon Bedrock Agent |
| RAG / Knowledge Base | Bedrock Knowledge Base + S3 |
| Guardrails | Bedrock Guardrails + LiteLLM content filters |
| Observability | OpenTelemetry → Grafana Cloud |
| Cost visibility | LiteLLM spend tracking + Infracost |
| Multi-cloud connectivity | HTTPS + API key / OIDC federation |
| IaC | Terraform (modular, destroy-safe) |
| CI/CD | GitHub Actions |

---

## Document Index

| # | Document | Description |
|---|---|---|
| 01 | [Requirements — FR & NFR](docs/01_requirements_fr_nfr.md) | All functional and non-functional requirements |
| 02 | [Use Cases](docs/02_use_cases.md) | Detailed use case specifications |
| 03 | [High-Level Design](docs/03_high_level_design.md) | Architecture, component map, data flows |
| 04 | [Technology Decisions (ADR)](docs/04_adr_technology_decisions.md) | Why each technology was chosen |
| 05 | [Cost Estimates](docs/05_cost_estimates.md) | PoC cost breakdown and destroy/recreate guidance |
| 06 | [Implementation Roadmap](docs/06_implementation_roadmap.md) | Phase-by-phase build plan |
| 07 | [Open Questions & Decisions Log](docs/07_open_questions.md) | Items requiring further decision |

---

## Quick Start (after full implementation)

```bash
# Clone
git clone <repo-url> && cd Aura_Promptly

# Estimate cost before deploying
cd infra && infracost breakdown --path .

# Deploy hub (Phase 1)
terraform -chdir=infra/hub init && terraform -chdir=infra/hub apply

# Destroy everything when not in use
terraform -chdir=infra/hub destroy
```

---

## Project Status

| Phase | Status | Description |
|---|---|---|
| Phase 0 | ✅ Complete | Requirements & HLD |
| Phase 1 | 🔲 Not started | Hub foundation (VPC, ECS, LiteLLM, Bedrock) |
| Phase 2 | 🔲 Not started | RAG pipeline + Bedrock Knowledge Base |
| Phase 3 | 🔲 Not started | Bedrock Agent |
| Phase 4 | 🔲 Not started | Azure spoke + sample app |
| Phase 5 | 🔲 Not started | GCP spoke + sample app |
| Phase 6 | 🔲 Not started | AWS spoke + sample app |
| Phase 7 | 🔲 Not started | Observability (OTel → Grafana) |
| Phase 8 | 🔲 Not started | OpenAI + Vertex AI integration |
| Phase 9 | 🔲 Not started | Guardrails hardening |
| Phase 10 | 🔲 Not started | CI/CD (GitHub Actions) |
