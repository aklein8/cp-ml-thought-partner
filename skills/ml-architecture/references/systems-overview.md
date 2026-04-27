# ClassPass ML Systems Architecture

## Organizational Context

The ML/Data Squad sits within CP Engineering. The squad's mission: enhance partner and user experiences within ClassPass by developing ML tools that integrate with engineering systems.

### Team
- **Engineering Manager**: Irfan Baig
- **Product Manager**: Ali Asif
- **Engineers**: Chris Starace, Fellipe Lamoglia, Carol Ransatto, Jonathan Wong, John O'Donnell, Clarisse Scofield
- **Slack**: #squad-ml-data / #squad-cp-ml-data-smartools
- **Kanban**: https://classpass.atlassian.net/secure/RapidBoard.jspa?rapidView=186

### Ownership Boundaries
- **Owns**: ML models, ML infrastructure, ML platform tooling, documentation for squad-originated models
- **Does NOT own**: Services that ingest ML model output, documentation for externally-originated models, alerts for external model input quality

## Two-Division Architecture

CP engineering splits into two architectures with different operational models.

### Engineering Division
- ~80 HTTP services communicating via JSON payloads
- Each service has its own database, often Redis cache, SQS topics/queues
- Written in Kotlin/Java, Python, or TypeScript/JavaScript
- Dockerized, running in ECS (cluster: `platform-Infra-production-permanent`)
- Workers run continuously processing queue data; Jobs run periodically via Airflow
- Swagger admin interfaces for most services
- Real-time, latency-sensitive

### P&I (Pricing & Inventory) Division
- Internal-facing only, not real-time (delays tolerable)
- Oriented around the Analytics Database (superset of all engineering databases)
- Data streams in via Fivetran or DMS
- BI adds many derived tables; giant batch system recomputed daily/hourly
- Jobs written in Python or SQL, data exchange via S3
- Most jobs run daily, some more frequently

## Infrastructure Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Compute | AWS ECS (Fargate) | Service hosting |
| Compute | EC2 (Parthenon instance) | Legacy batch jobs (Affinity, crontabs) |
| Data Warehouse | Snowflake | Primary analytical store (cp_mldata, cp_pricing, cp_prod schemas) |
| Analytics DB | PostgreSQL (legacy) | Historical analytical store, being migrated to Snowflake |
| Cache | Redis (ElastiCache) | CPA price quotes, base prices, PriceMatch adjustments |
| Object Storage | S3 | Data exchange between P&I jobs, model artifacts, SmartSpot output |
| Messaging | SQS/SNS | Event-driven processing (SmartSpot predictions, HAC/F&B tagging events) |
| Orchestration | Airflow | Job scheduling (airflow.classpass.engineering) |
| Data Transforms | DBT (Rednose) | SQL-based data transformations in Snowflake |
| Data Ingestion | Fivetran | Syncing production databases to Snowflake |
| ML Tracking | MLflow | Model experiment tracking and prompt management |
| Notebooks | AWS SageMaker | Model development and experimentation |
| Monitoring | New Relic, Datadog | APM, dashboards, logging |
| Alerting | OpsGenie | On-call alerting |
| Deployment | Internal Deployer tool | ECS service deployment |
| CI/CD | Jenkins (most), local Docker builds (Pricing) | Build and deploy pipelines |

## Key Repositories

### ML Squad Owned
| Repo | Language | Purpose |
|------|----------|---------|
| `classpass/affinity` | Java | Schedule recommender (P&I) |
| `classpass/credit-price-api` | Kotlin | Real-time pricing service |
| `classpass/classpass-smartspot` | Python | Fill prediction model |
| `classpass/supermodels` | Python/Kotlin | Multi-model service (RTR, LCM, Churn, Similar Venues) |
| `classpass/churn` | Python | Churn model training code |
| `classpass/recommender` | Python | Recommender models (HAC, F&B tagger live here) |
| `classpass/rednose` | SQL/Python | DBT project for data transforms |
| `classpass/ml-flow` | Python | MLflow deployment |
| `classpass/ml-utils` | Python | Shared ML utility library |
| `classpass/ml-base` | Docker | Base Docker image for ML services |
| `classpass/data-importer` | Python | Data import library |
| `classpass/protobuf-definitions` | Protobuf | Shared protobuf schemas (used by CPA) |
| `classpass/ml-ddd-template` | Template | Domain-driven design template for new ML services |
| `classpass/ml-exploration` | Jupyter | Hackathon notebooks |

### Partially Owned / Supported
| Repo | Purpose |
|------|---------|
| `classpass/pricing` | Dynamic pricing job (P&I) |
| `classpass/airflow` | Airflow DAG definitions |
| `classpass/docker-images` | Shared Docker images |
| `classpass/analytics-db` | Analytics ETL and derived tables |
| `classpass/spot-recs-service` | Spot Recommendations (downstream of SmartSpot, owned by another squad) |

## Data Flow Patterns

### Real-Time Path
```
User Request → Availability/Search → CPA (Redis lookup) → Price Quote
                                      ↑
                              Pricing Job → S3 → Price Loader Worker → Redis
```

### Batch ML Path
```
Snowflake/Analytics DB → Feature Engineering → Model Training (SageMaker/Local)
    → Model Artifacts (S3/MLflow) → Batch Inference (Airflow/ECS) → Snowflake Output
```

### Event-Driven Path (Tagging)
```
Class/Venue Update → SQS Event → HAC/F&B Worker → LLM (OpenAI) → Postgres → Snowflake
```

### SmartSpot → Pricing Path
```
SmartSpot (batch, Parthenon) → S3 → SNS → Spot Recs Service → SQS → Spot Adjustments
                                          ↓
                                    Pricing Job (uses SmartSpot fill predictions for Protection)
```

## Snowflake Schema Map

| Schema | Purpose |
|--------|---------|
| `CP_MLDATA.RECOMMENDER` | LLM tagging output, HAC predictions |
| `CP_MLDATA.CHURN` | Churn model predictions and snapshots |
| `CP_MLDATA.MLDATA_PROD` | Feature store, churn snapshots |
| `CP_MLDATA.RTR_BATCH` | RTR fraud detection output |
| `CP_ML_DATA_SMARTTOOLS.SKU_PRICING` | SKU pricing models |
| `CP_ML_DATA_SMARTTOOLS.RTR_BATCH` | RTR batch (new schema, pending migration) |
| `CP_PRICING.AFFINITY` | Affinity recommendation output |
| `CP_PRICING.PRICING` | Pricing configuration (MSA settings, overrides) |
| `CP_PROD.RECOMMENDER_PUBLIC` | HAC classifier predictions |
| `CP_PROD.CHURN_PUBLIC` | Churn predictions (Fivetran sync) |
| `CP_PROD.TAGS_PUBLIC` | Content tags |

## Design Principles

### ML Domain Driven Design
1. **Separate repos per service domain** — each repo hosts models with the same business purpose, single Dockerfile per repo
2. **Shared code as artifacts** — shared utils live in artifactory with robust testing; consumers responsible for behavioral tests
3. **Rule of 3 for generalization** — don't abstract until three copies exist; cost of maintenance outweighs refactoring risk at two copies

### Production ML System Requirements
Every production ML system must have: version control, documentation (runbooks, model description, I/O, business purpose, POC), CI/CD, automated testing, alerting/logging, model monitoring, system monitoring, data quality checks, model governance, automated retraining, security, automated inference (batch and/or real-time).

## Migration Status (as of 2024 lookback)
- **95% Snowflake migration** complete (Pricing, SmartSpot, ML services all migrated)
- **75% P&I models** migrated from PI2 to DBT (Rednose)
- **Premium pricing** sunset (Dec 2024) — 20% runtime reduction
- **ML Platform 1.0** decommissioned (March 2024)

## SharePoint TDDs (Not Yet Ingested)
These technical design documents are referenced in Notion but stored in SharePoint. They should be uploaded for deeper architectural context:
- ML-PLAT_SupervisedModelOnboarding.docx (Supermodels TDD)
- TDD - Targeted User Campaigns.docx (STO/TUC)
- TDD - Hierarchical Agentic Classifier (HAC)
- TDD - F&B Autonomous Venue Tagger
- Pricing Job Migration from Data Pipeline.docx
- [DACI] N+2 Tags Expansion.docx
