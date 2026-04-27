# ClassPass ML Models Catalog

## Active Models

### 1. Churn Model (Consumer Churn Prediction)

**Type**: Batch-prediction supervised ML model (XGBoost)
**Purpose**: Predicts subscriber propensity to churn (Low/Medium/High) for proactive save campaigns
**Owner**: ML Squad (model developed by John O'Donnell)
**Service**: Supermodels microservice
**Goal**: Help lifecycle marketing reduce churn by 1% with proactive save campaigns

#### Training
- Algorithm: XGBoost
- Training code: `classpass/churn` repo → `model_development/sub_retention_sample.ipynb`
- Features: User-level features documented in DBT feature store (`production-platform-rednose.classpass.com/#!/model/model.rednose.john_model_features`)
- Feature store: Always reads `mldata_prod` features schema in Analytics (even from dev)
- Retraining: Automated via Airflow DAG `churn-retrain` (Fargate task)

#### Data Flow
```
User Features (Snowflake/Analytics) → XGBoost Model → Predictions
    → churn.predictions (Analytics DB)
    → cp_prod.churn_public.predictions (Fivetran sync)
    → cp_mldata.churn.churn_predictions (copy)
    → cp_mldata.mldata_prod.churn_predictions_snapshot (SCD2 snapshot)
```

#### Output Tables
| Table | Description |
|-------|-------------|
| `churn.predictions` | Latest daily run (user_id, prediction 0-1, level Low/Med/High, last_updated) |
| `cp_prod.churn_public.predictions` | Fivetran sync of above |
| `cp_mldata.churn.churn_latest_predictions` | Current valid data from snapshot |
| `cp_mldata.churn.churn_monthly_metrics` | Monthly aggregated metrics |
| `cp_mldata.churn.churn_previous_day_predictions` | Previous day for comparison |
| `cp_mldata.churn.churn_users_churned` | Users who have churned |

#### Monitoring
- Tableau: ChurnDashboard (tracks model performance over time)
- New Relic: Retrain logs (`production-platform-Infra-permanent-churn-churn-retrain-fargate`)
- Consumers: Lifecycle Marketing team

### 2. Late Cancellation Model (LCM)

**Type**: Real-time supervised ML model
**Purpose**: Predicts whether a user will late cancel (within 12 hours of class start)
**Owner**: ML Squad
**Service**: Supermodels microservice
**Impact**: $4,500+/day in savings via reservation deferral

#### How It Works
- Predictions used by Reservation Service for the reservation deferral process
- When model predicts high late-cancel probability, reservation may be deferred (held rather than confirmed)
- This reduces lost spots and partner dissatisfaction from no-shows

#### Architecture
- Feature engineering and model architecture documented in dedicated Notion page
- Separate LCM Runbook for operations
- Launch gate doc covers rollout plan and safety checks

### 3. RTR Real-Time (Repeat Trial Redemption)

**Type**: Real-time SQL-based heuristic model
**Purpose**: Identifies fraudulent users during signup who repeatedly create accounts for free trials
**Owner**: ML Squad
**Service**: Supermodels microservice
**Impact**: 10% → 80%+ deflection rate (2024)

### 4. RTR Batch Model

**Type**: Batch fraud detection system (DBT + Python)
**Purpose**: Post-signup detection of trial abuse via 17 matching rules + network analysis
**Owner**: ML Squad (John O'Donnell)
**Cadence**: Every 30 minutes

#### Architecture
```
Incremental Sources (user devices, locations, promos, status)
    → User Feature Collection (user_feature_stg)
    → 17 Matching Rules (phone, name+CC, IDFA, email, location, etc.)
    → Feature Aggregation (search) + Scoring (search_matches)
    → Logging (matches_log) + Filtering (matches_filtered)
    → Network Analysis (connected components via NetworkX)
    → Reservation Enrichment (rtr_res_to_payout)
```

#### 17 Matching Rules Summary
Strong signals: phone matches, name+CC, email+CC, blocked cards+names, IDFA matches
Medium signals: name+IDFA (exact and fuzzy), email+IDFA (exact and fuzzy), CC-only (threshold n>=2)
Conditional (when device data unavailable): device_model+name, name+lat/long, email+lat/long
Uniqueness-gated: full_name_counts, email_counts (require venue overlap concentration)

#### Network Analysis
- Graph-based: each user is a node, matches are edges
- NetworkX finds connected components = fraud clusters
- Original account = earliest trial start in component
- Output: cluster_id, original_user_id, is_original, cluster_size, account_rank

#### Cluster Triage (Semi-Automated)
7 fraud type tags: venue_collusion, partner_payout_anomaly, fnb_resale, serial_abuse, billing_failure, corp_exploit, currency_arbitrage
3 status tags: still_bleeding, dead_trial_satellite, internal_test
Investigation workflow produces markdown reports with Tier 1 evidence, loss attribution, mechanism proof, calibrated confidence, recommended action.

#### Partner Payout Reconciliation
`rtr_res_to_payout` identifies FCF reservations where ClassPass should retroactively pay partners because the user was a repeat trialer. `pay_partner_flag` = true when: not original account, was FCF, repeat venue visit within cluster, flagged within 30 days.

#### Guardrails
- `rtr_batch_config.max_new_users = 3000` safety limit per run (prevents mass-flagging)
- Corporate plan segregation on all matching rules
- Internal email exclusion (@classpass.com, @playlist.com, @mindbody.com)
- Allow list for known false positives

### 5. Affinity Recommender

**Type**: Batch recommendation model
**Purpose**: Generates 300 schedule recommendations per active user
**Owner**: ML Squad (runs on Parthenon)
**Cadence**: Every 6 hours (4x/day)
**Consumers**: Pricing Job (recommended discounts), dynamic pricing, SmartSpot, discounted schedules

### 6. Similar Venues Recommender

**Type**: Batch recommendation
**Service**: Supermodels (Airflow DAG: `similar-venues-recommendations`)
**Purpose**: Recommends similar venues to users

### 7. SmartSpot

**Type**: Batch prediction model
**Purpose**: Predicts studio fill rates to determine ClassPass spot allocation
**Consumers**: Pricing Job (protection), Spot Recs (spot allocation)
**See**: ml-pricing-engine skill for detailed architecture

## Retired/Decommissioned Models

### STO/TUC (Send Time Optimization / Targeted User Campaigns)
- **Disabled**: Airflow 2024, Rednose 2025
- **Purpose was**: Personalized send times for Braze campaigns using contextual bandit model
- **Algorithm**: VW (Vowpal Wabbit) bandit model
- **Feature store**: Read from mldata_prod features schema
- **Repository**: `classpass/sto`

### ML Platform Service (Tags Predictor v2)
- **Decommissioned**: March 2024
- **Repository**: `classpass/ml-platform-service`
- **Replaced by**: LLM-based tagging system

## Active Projects (2026)

### P&I Model Migration
Migrating remaining P&I models from PI2 (legacy) to DBT/Rednose:

| Model | Status | Complexity | Owner | Type |
|-------|--------|-----------|-------|------|
| RTR Hourly/Daily | Planned | Medium | John | SQL |
| Proteus/Recommended Plans | Planned | Very High | Leang | - |
| comp_scoring_wellness_prospects | Onboarded | Low | Shreya | SQL |
| vls_predict_inbound_w2 | Planned | High | Leang | - |
| user location model automation | Onboarded | Medium | Shreya | SQL |
| wellness_baseline_points | Onboarded | Medium | Shreya | Python+SQL |
| Rolling Usage Count | Onboarded | High | Leang | Python+SQL |
| corporate_net_rev_prediction_inbound | Onboarded | High | Leang | Python+Pkl+SQL |
| cp_value_matrix | Onboarded | Medium | Shreya | Python+SQL |
| food_bev_scoring_snf | Onboarded | High | Shreya | Python+Pkl+SQL |
| churned venue value model | Onboarded | Medium | Shreya | SQL |
| Early Life Recommender | Unplanned | Medium | Ethan | - |

### Finance Churn Analysis
Active analytical project exploring churn patterns for finance stakeholders.

### SKU Pricing
New DBT pipeline for SKU item/modifier pricing (see ml-pricing-engine skill).

## Model Development Infrastructure

### MLflow
- Tracks ML model parameters and runs
- Production: `production-platform-ml-flow.classpass.com`
- Development: `development-use1-ml-flow.classpass.com`
- Also used for LLM prompt management (F&B tagger prompts)
- Single instance per environment (not critical path)
- Repository: `classpass/ml-flow`

### SageMaker Notebooks
- Used for model experimentation
- Access via Okta → AWS → SageMaker → Notebook Instances
- Best practices: auto-shutdown after 1 hour idle, proper naming by squad
- VPC access to Analytics DB available on request
- Git integration for notebook repos

### Rednose (DBT Runner)
- Repository: `classpass/rednose`
- DBT project for all data transformations
- Runs models in Snowflake
- Feature store documentation: `production-platform-rednose.classpass.com`
- Airflow DAGs: `rednose_rtr` (daily), `rednose_rtr_hourly`, `rtr-batch` (hourly)

### Production ML Checklist
Every productionized model must have: version control, documentation (runbooks, model description, I/O, business purpose, POC), CI/CD, automated testing, alerting/logging, model monitoring, system monitoring, data quality checks, model governance, automated retraining, security, automated inference.

## 2024 Impact Summary

| Model/System | Impact |
|-------------|--------|
| RTR (Real-Time + Batch) | 80%+ trial fraud deflection rate |
| LLM Tagging | +200 bps T2S conversion rate in LFC onboarding |
| Late Cancellation Model | $4,500+/day savings |
| Overall ML Services | $2M+ direct business impact |
| Snowflake Migration | 95% complete |
| P&I → DBT Migration | 75% of models |
| Premium Pricing Sunset | 20% pricing runtime reduction |
