---
name: ml-models-lifecycle
description: >
  Comprehensive catalog of all ClassPass ML models including their architectures, training pipelines,
  feature stores, monitoring, and operational details. Use this skill when someone asks "what ML models
  does ClassPass have", "how does the churn model work", "explain the RTR fraud model",
  "what is the late cancellation model", "how does Affinity work", "what features does the churn model use",
  "how is RTR batch different from RTR real-time", "what is cluster triage", "fraud detection architecture",
  "model training pipeline", "feature store", "model monitoring", "what models are active",
  "what models are retired", "similar venues recommender", "supermodels service",
  or any question about specific ML model implementations, their data flows, training processes,
  feature engineering, output tables, consumers, or operational runbooks.
version: 0.1.0
---

# ClassPass ML Models & Lifecycle

Load `references/models-catalog.md` for the complete models catalog.

## Quick Orientation

ClassPass runs 7+ active ML models across 4 service deployments:

### Supermodels Service (multi-model, ECS)
- **RTR Real-Time**: Heuristic fraud detection during signup (80%+ deflection)
- **Late Cancellation Model**: Predicts late cancels, powers reservation deferral ($4.5K/day savings)
- **Churn Model**: XGBoost subscriber churn prediction (Low/Med/High), feeds lifecycle marketing
- **Similar Venues**: Batch venue recommendations

### Standalone Services
- **Affinity**: Schedule recommender (300 recs/user, 4x/day on Parthenon)
- **RTR Batch**: DBT + Python fraud detection (17 matching rules, network analysis, 30-min cadence)
- **SmartSpot**: Fill prediction for studio classes (feeds Pricing and Spot Recs)

### Retired
- **STO/TUC**: Send time optimization (disabled 2024-2025)
- **ML Platform Service**: Legacy tagging (decommissioned March 2024)

## When Answering Model Questions

1. Read `references/models-catalog.md` for the full catalog
2. Identify the model and its service deployment (Supermodels vs standalone)
3. Understand the inference pattern (real-time API vs batch prediction)
4. Trace the feature → training → inference → output → consumer pipeline
5. Note operational details: cadence, monitoring, alerting, retraining schedule
6. For RTR Batch specifically, understand the 17 matching rules, network analysis (connected components), and cluster triage workflow

## Key Patterns

- **Feature store**: `mldata_prod` schema in Analytics/Snowflake (shared across Churn, STO, RTR)
- **Model tracking**: MLflow for experiment tracking and prompt management
- **Batch inference**: Airflow-orchestrated, writes to Snowflake
- **Real-time inference**: ECS services behind internal load balancers
- **Retraining**: Automated via Airflow DAGs (Churn retrain, RTR hourly/daily)
- **Monitoring**: Combination of Tableau dashboards (Churn), Datadog (RTR, services), New Relic (logs, APM)

## 2024 Impact
- RTR: 80%+ trial fraud deflection
- LCM: $4,500+/day savings
- Churn: Powers lifecycle marketing save campaigns
- Overall: $2M+ direct business impact across all ML services
