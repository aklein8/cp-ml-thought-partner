# ClassPass ML Repository Technical Details

## 1. Churn Model (`classpass/churn`) — Python

### Purpose
Predicts customer churn risk and generates similar venue recommendations using ML models.

### Architecture
```
churn/
├── src/
│   ├── api/                          # API layer
│   │   ├── controllers/
│   │   │   ├── health_controller.py  # /healthCheck endpoint
│   │   │   ├── churn_controller.py   # /predict, /retrain, /init endpoints
│   │   │   └── similar_venues_controller.py  # /svm/run_model endpoint
│   │   └── dto/                      # Request/response data transfer objects
│   ├── domain/
│   │   ├── managers/
│   │   │   ├── churn_manager.py      # Orchestrates batch predictions & retraining
│   │   │   └── similar_venues_manager.py  # Orchestrates venue rec workflow
│   │   ├── engines/
│   │   │   ├── churn_prediction_engine.py    # One-hot encoding, batch prediction
│   │   │   ├── churn_retraining_engine.py    # Data cleaning, train/test split, metrics
│   │   │   ├── model_config_load_engine.py   # Loads model config from MLflow
│   │   │   └── similar_venues_recommendation_engine.py  # Similarity scoring, recs
│   │   └── monitoring/
│   │       ├── papertrail.py         # Papertrail logging setup
│   │       └── newrelic_support.py   # New Relic integration
│   └── providers/
│       ├── churn_feature_store_provider.py   # Reads from Analytics feature store
│       ├── churn_prediction_store_provider.py  # Writes predictions to DB
│       ├── churn_model_provider.py           # MLflow model load/upload/metrics
│       ├── similar_venues_feature_store_provider.py
│       ├── similar_venues_recommendation_store_provider.py
│       ├── similar_venues_s3_provider.py     # S3 upload for recs
│       └── similar_venues_mlflow_provider.py # MLflow metrics logging
└── tests/
    ├── test_churn_prediction_engine.py
    └── test_churn_retrain_engine.py
```

### Key Flows

**Prediction Flow** (`ChurnManager.predict_user_churn()`):
1. Query all users from feature store table
2. Load model configs (features required) from MLflow
3. Split users into groups, query features per group
4. Run `ChurnPredictionEngine` predictions per group
5. Write predictions to prediction store (Postgres → Snowflake)

**Retraining Flow** (`ChurnManager.retrain_churn()`):
1. Query all users and feature data in batches
2. Clean and preprocess data per batch
3. Concatenate into large DataFrame
4. Retrain XGBoost model via `ChurnRetrainingEngine`
5. Upload model parameters + metrics to MLflow

**Similar Venues Flow** (`SimilarVenuesManager.run_model()`):
1. Query venue features and zip neighbor data
2. Generate recs via `SimilarVenuesRecommendationEngine`
   - `train_similarity()`: dimensionality reduction + venue correlation within market/country
   - `calculate_scores()`: similarity + geographic modifiers
   - `get_recommendation()`: full pipeline → `SimilarVenueRecommendation` objects
3. Log metrics to MLflow, upload to S3, write to recommendation store

### Data Providers
- **Feature Store**: Reads from Analytics DB `mldata_prod` features schema
- **Prediction Store**: Writes to `churn.predictions` (Postgres)
- **Model Store**: MLflow for model artifacts, parameters, and retrospective metrics
- **S3**: Similar Venues recommendations uploaded to S3

---

## 2. Credit Price API (`classpass/credit-price-api`) — Kotlin

### Purpose
Real-time pricing service surfacing credit prices to users. The most critical ML-adjacent service.

### Architecture
- **Framework**: Hybrid Jersey + ktor (problematic — see modernization)
- **Primary data store**: Redis (ElastiCache) — not a cache, the actual price store
- **Config**: Mix of config file and environment variables

### Key Components
- **Service**: Handles price quote requests from Availability
- **Price Loader Worker**: Continuously imports Pricing Job output from S3 → Redis
- **PriceMatch Import**: Airflow DAG imports hourly adjustments

### CPA Modernization (2025) — Known Problems
1. **Redis Client**: Lettuce library causes timeouts/pool exhaustion on new containers during autoscale/deploy. Only CP service using this fully-async library.
2. **Jersey + ktor Hybrid**: Makes onboarding hard, config is split. Plan: remove ktor, rebase on Dropwizard/Jersey.
3. **Coroutines**: New Relic can't trace them properly. Under load, fully-async coroutines don't respond to backpressure → unbounded memory growth → crash. Plan: remove coroutines or move all to IO dispatcher.
4. **Redis as Primary Store**: CPA uses Redis uniquely — as primary data store, not cache. Three Redis pathways: price loading, quote generation, quote verification. Could use secondary Redis for read operations.
5. **Prometheus**: JMX/Prometheus integration incomplete — Jersey metrics not exported, can't autoscale on connection count.
6. **Async Code Generally**: High traffic + async = unbounded internal queue when upstream slows. Needs bulkhead limits.

### CPA Quote Flow
```
User Request → Availability → CPA /v6/credits/check/{userId}
    → Fetch base prices from Redis
    → Apply segmented pricing rules (from Segments service)
    → Calculate per-user price
    → Save quote to Redis (600s TTL)
    → Return quote to Availability
```

### CPA Check Flow
```
Reservation Request → CPA check endpoint
    → Fetch saved quote from Redis
    → Validate price still valid
    → Return validation result
```

### Related Repos
- `classpass/jade` — Contains KonfigLettuce Redis client code
- `classpass/protobuf-definitions` — Protobuf schemas for CPA communication

---

## 3. Affinity (`classpass/affinity`) — Java

### Purpose
Collaborative filtering recommendation engine generating 300 personalized schedule recommendations per active user.

### Architecture
```
affinity/
├── src/main/java/com/classpass/affinity/
│   ├── model/
│   │   ├── UserEntry.java          # User entity (country, market, attributes)
│   │   ├── ReservationEntry.java   # Reservation (type, classId, trainerId, venueId, timestamps)
│   │   ├── VenueEntry.java         # Venue attributes
│   │   └── ActivityEntry.java      # Activity attributes
│   └── recommendation/
│       ├── ETL.java                # Extracts data from DB in chunks → JSON files
│       ├── ProcessData.java        # Transforms JSON → features (HashMaps, Tables, Lists)
│       ├── Trainer.java            # Iterative model training
│       ├── Model.java              # Model parameters
│       └── Recommender.java        # Scoring + ranking → top recommendations
├── src/reporting/
│   └── extract-spotmatch-weeksids.v3.pl  # Raw data extraction scripts
└── postal_clusters/
    └── postal_clusters_gls.top.py  # Geographic clustering for location-aware recs
```

### Pipeline
1. **ETL** (`ETL.java`): Read from PostgreSQL in chunks → JSON files (reservations, venues, activities, users)
2. **Process** (`ProcessData.java`): Parse JSON → generate features (venue, activity, start hour, period) → HashMaps/Tables
3. **Train** (`Trainer.java`): Initialize model → evaluate → mutate → iterate
4. **Recommend** (`Recommender.java`): Score candidates using trained model → rank → output top 300 per user
5. **Location**: `postal_clusters` groups postal codes for geographic relevance

### Operations
- Runs on Parthenon (EC2) from Chris's crontab, every 6 hours
- Output: Protobuf to S3 + Snowflake affinity tables
- Memory-intensive (dedicated New Relic memory monitoring)

---

## 4. Recommender (`classpass/recommender`) — Python

### Purpose
Houses the HAC (Hierarchical Agentic Classifier) and F&B Auto Tagger workers. The newer recommendation/classification service.

### Key Components
- **HAC Worker**: SQS listener for class update events → preprocessing → OpenAI LLM → L1/L2/L3 tag predictions → Postgres
- **F&B Auto Tagger Worker**: Same pattern but filtered to F&B venues only
- **Preprocessing Engine**: Text cleaning/normalization, taxonomy glossary pull from Snowflake
- **LLM Integration**: OpenAI-based classification
- **Database**: Postgres for predictions, replicated to Snowflake

### Monitoring
- Datadog: `recommender-hierarchical-tagging-worker` and `recommender-tagging-worker`
- New Relic: Application performance and error tracking

---

## 5. Rednose (`classpass/rednose`) — SQL/Python (DBT)

### Purpose
The DBT project that runs all SQL-based data transformations in Snowflake. The backbone of ML data processing.

### Key Model Areas
- **Churn features**: `churn_predictions`, `churn_predictions_snapshot`, `churn_monthly_metrics`
- **Late cancellation features**: `late_cancel_features` (training data), `*_live` tables (inference)
- **RTR Batch**: Full fraud detection pipeline (17 matching rules, network analysis, cluster triage)
- **SKU Pricing**: `priced_skus`, `sku_credit_amounts`, `sku_pricing_report`
- **P&I models**: Various migrated models (venue value, wellness scoring, etc.)

### Airflow DAGs
- `rednose_rtr` (daily), `rednose_rtr_hourly`, `rtr-batch` (hourly)
- `churn-retrain` (Fargate)
- `similar-venues-recommendations`

### Documentation
- DBT docs hosted at `production-platform-rednose.classpass.com`
- Feature store documentation available there

---

## 6. SmartSpot (`classpass/classpass-smartspot`) — Python

### Purpose
Predicts studio class fill rates to determine ClassPass spot allocation.

### Predictor Internals
**Glossary**:
- **State**: Model used for predictions (S3, one file per venue)
- **Output**: Predictions (S3, one file per venue, JSON lines per schedule)
- **Messages**: Individual actions (Reservation, ScheduleCapacity) stored in database

**Processing Architecture**:
```
SmartSpot Service
    → Primary Executor (thread pool)
        → Process Venue 1..N in parallel
            1. Read State from S3
            2. Read new Messages from DB since last state
            3. Update State with new messages
            4. Generate Predictions
    → Secondary Executor (thread pool)
        → Upload State to S3
        → Upload Output to S3
```

**Key Design**: Venue-oriented processing — processes all schedules for one venue together. Can filter via whitelist, blacklist, or partition.

---

## 7. ML Utilities (`classpass/ml-utils`) — Python

### Purpose
Shared utility library consumed by multiple ML services. Published as an artifact.

### Usage Pattern
Per DDD principles: consumers are responsible for behavioral tests ensuring compatibility with util changes.

---

## 8. ML Docker Base Image (`classpass/ml-base`) — Docker

### Purpose
Base Docker image for all ML services. Provides common Python dependencies, system libraries, and configuration.

### Usage
Referenced as base image in Dockerfiles across ML repos (churn, recommender, etc.).

---

## 9. ML Data-Driven Domain Template (`classpass/ml-ddd-template`) — Template

### Purpose
Scaffolding template for new ML services following Domain-Driven Design principles.

### Design Principles Encoded
1. Separate repos per business domain (single Dockerfile)
2. Shared code via artifacts (ml-utils)
3. Rule of 3 for generalization

---

## 10. Protobuf Definitions (`classpass/protobuf-definitions`) — Protobuf

### Purpose
Shared protobuf schema definitions used for CPA inter-service communication and Affinity output format.

### Usage
- CPA uses protobuf for efficient price data serialization
- Affinity outputs recommendations in protobuf format to S3

---

## 11. ML Platform Service (`classpass/ml-platform-service`) — Legacy

### Purpose
Legacy ML-driven class tagging service. **Decommissioned March 2024**.

### Historical Context
Was the first attempt at automated class tagging. Replaced by the LLM-based tagging approach (LLM Tagger → HAC).

---

## 12. ML Exploration (`classpass/ml-exploration`) — Jupyter

### Purpose
Hackathon notebooks and experimental ML work. Not production code.

### Notable Contents
- Hackathon 2022-12 notebooks
- Hackathon 2024-12: LLM Assistant + Affinity experiments

---

## 13. ML Interview Notebooks (`classpass/ml-interview-notebooks`) — Jupyter

### Purpose
Standardized notebooks used for ML engineer candidate interviews.

---

## 14. Late Cancellation Model — Feature Engineering Details

### Feature Categories

**Intrinsic Features** (static properties of reservation/user):
1. Class day of week
2. Reservation created day of week
3. Class time of day (bucketed)
4. Reservation created time of day (bucketed)
5. Application type (iOS/Android/Desktop)
6. Booking lead time (minutes)
7. Source
8. MSA
9. Plan supercategory

**Behavioral Features** (probabilities from past actions, 1/3/6 month windows):
1. User cancel probability
2. MSA cancel probability
3. Venue cancel probability
4. Class cancel probability
5. Reservation created hour cancel probability
6. Class-user cancel probability (cross)
7. Venue-user cancel probability (cross)
8. Reservation created hour-user cancel probability (cross)
9. Class day of week-user cancel probability (cross)
10. Class day of week-class cancel probability (cross)

### Feature Pipeline (DBT/Rednose)
- **Training data**: `late_cancel_features` table — historical data with all features
- **Inference data**: `*_live` tables — most up-to-date features, replicated daily to Supermodels DB
- DAG documented at: `production-platform-rednose.classpass.com/#!/model/model.rednose.late_cancel_features`
