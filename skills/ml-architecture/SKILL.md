---
name: ml-architecture
description: >
  Deep knowledge of ClassPass ML engineering architecture, service dependencies, infrastructure,
  and deployment patterns. Use this skill when someone asks "how does the ML system work",
  "what services does ML own", "where does this data flow", "what's the tech stack",
  "how is X deployed", "what repos does ML own", "explain the P&I vs Engineering split",
  "what Snowflake schemas exist", "how do services communicate", or any question about
  the technical architecture, infrastructure, or systems engineering of the ClassPass ML ecosystem.
version: 0.1.0
---

# ClassPass ML Architecture

Load `references/systems-overview.md` for the systems map, infrastructure, and Snowflake schemas.
Load `references/repo-technical-details.md` for deep code-level documentation of all 14 repos — directory structures, key classes, data flows, API endpoints, and architectural patterns.

## Quick Orientation

The ML/Data Squad builds and maintains ML models and infrastructure within ClassPass. The engineering landscape splits into two divisions:

**Engineering**: ~80 real-time HTTP services (Kotlin/Java, Python, TS) running in ECS, communicating via JSON, each with its own DB + Redis + SQS.

**P&I (Pricing & Inventory)**: Internal batch systems oriented around Snowflake (formerly Analytics DB), running Python/SQL jobs via Airflow, exchanging data via S3.

## When Answering Architecture Questions

1. Read `references/systems-overview.md` for the full systems map
2. Identify which domain the question falls into (pricing, recommendations, classification, fraud, lifecycle, platform)
3. Trace the data flow from source → processing → output → consumer
4. Note the deployment pattern (ECS service vs Airflow batch vs Parthenon crontab)
5. Call out monitoring touchpoints (New Relic, Datadog, OpsGenie)

## Key Architectural Facts

- **14+ repos** owned by ML Squad, plus several partially owned
- **Snowflake** is the primary data warehouse (95% migrated from Analytics DB/Postgres)
- **Airflow** orchestrates all batch jobs
- **Redis** (ElastiCache) backs real-time pricing via CPA (~171M keys)
- **DBT (Rednose)** handles SQL-based data transformations
- **MLflow** tracks model experiments and manages LLM prompts
- **Parthenon** (shared EC2 instance) still runs Affinity and SmartSpot crontabs

## Design Principles

1. Separate repos per service domain (single Docker file per repo)
2. Shared code as artifacts with robust testing
3. Rule of 3 for code generalization
4. Every production ML system must meet the production checklist (version control, docs, CI/CD, testing, monitoring, etc.)

## SharePoint TDDs to Request

For deeper architectural context, these TDDs should be uploaded:
- ML-PLAT_SupervisedModelOnboarding.docx
- TDD - Targeted User Campaigns.docx
- TDD - Hierarchical Agentic Classifier (HAC)
- TDD - F&B Autonomous Venue Tagger
- Pricing Job Migration from Data Pipeline.docx
