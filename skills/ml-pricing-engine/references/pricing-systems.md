# ClassPass Pricing & Inventory Systems

## System Overview

The pricing domain is the most interconnected part of the ML ecosystem. It determines the credit price for every schedule and SKU in the ClassPass catalog, handles dynamic pricing, spot allocation, and partner payouts. The key systems are:

1. **Pricing Job** — Batch pipeline calculating prices for all schedules
2. **Credit Price API (CPA)** — Real-time service surfacing prices to users
3. **SKU Pricing** — DBT pipeline for SKU item/modifier pricing
4. **SmartSpot** — Fill prediction model for studio classes
5. **Spot Recommendations** — Downstream consumer of SmartSpot, manages spot allocation
6. **PriceMatch** — Adjustment system imported into CPA hourly

## Pricing Job

### Purpose
Calculates prices for every schedule (30M+ and rising) using dynamic pricing logic. Runs as a batch job in Fargate via Airflow.

### Repository
`classpass/pricing` (Python)

### Environments
| Environment | Cadence | Purpose |
|-------------|---------|---------|
| `production` | Every 2 hours | Full venue pricing |
| `fastprod` | Every 20 minutes | Recently-changed venues only |
| `staging` / `staging2` | On-demand | QA (reads prod data, writes to unused buckets) |

### Pipeline Components
1. **Generate dynamic premium prices** — deprecated as of Dec 2024
2. **Network prices** — main pricing pipeline
3. **Update credits** — most important and time-consuming component
4. **Generate static network payouts**
5. **Lookup settings data** — venue settings for pricing categorization
6. **Recommended discounts first run** — joins pricing with Affinity output
7. **Calculate historic fill** — protection logic using SmartSpot data
8. **Minimum payout lookup** — gets min payout from `partner-pay-ctrl` service
9. **Report** — output summary for monitoring

### Key Concepts
- **Protection**: After dynamic pricing, the system checks current and expected fill. As class approaches being full, minimum price rises toward retail rate.
- **SmartRate / AutoRate / Static / OnePool**: Different pricing modes for venues
- **Dynamic pricing**: Uses SmartSpot fill predictions to adjust prices based on demand

### Data Sources
- Analytics DB derived tables: `retail_prices_v2`, `retail_prices_v2_recent_update`, `hist_cp_fill`
- Pricing MSA Settings (Google Sheet → ETL → `pricing_and_inventory.*`)
- Venue-specific data via P&I Fivetran Integration

### Deployment
Local Docker build (not Jenkins). `make deploy-production deploy-fastprod` builds, pushes to ECR, tags. Airflow picks up image by tag.

### Monitoring
- Datadog Dashboard for pipeline metrics
- Pricing alert monitors auto-alert #pricing channel
- Report component output for QA (compare trends, histograms, averages)

### Potential Improvements
- Parallelize via multiple pipelines (currently only using 1 CPU on 12+ CPU machines)
- Replace pickle with cPickle (80%+ of fetch() time is unpickling)
- Replace json with ultrajson (70% of fetch_jsonl_zstd time is JSON decoding)
- Pull statically-priced schedules in real-time from CPA instead of batch

## Credit Price API (CPA)

### Purpose
Real-time pricing service. Surfaces prices to users via search (called by Availability service). Stores prices in Redis, applies per-user pricing rules.

### Repository
`classpass/credit-price-api` (Kotlin)

### Components
- **Service**: Provides real-time price quotes per user per schedule
- **Price Loader Worker**: Imports prices from Pricing Job S3 output into Redis constantly
- **PriceMatch Import Job**: Airflow DAG `pricematch_import_to_cpa_dag` imports adjustments hourly

### Redis Stores

#### CPRICE Redis
| Prefix | Description | Volume | TTL |
|--------|-------------|--------|-----|
| `const-price-v1-sid-*` | Constant prices | ~9K keys | 20 days |
| `schedule:*:user:*:pricedScheduleV2:raw` | User price quotes | ~1M records | 600 seconds |
| `base-price-v2--NETWORK-*` | Network base prices | Bulk of ~171M keys | 15 days |
| `base-price-v2--PREMIUM-*` | Premium base prices | Part of 171M keys | 15 days |
| `livestream-price-v1-venueId` | Livestream prices | Very small | No TTL |

#### PMATCH Redis
| Prefix | Description |
|--------|-------------|
| `pm-rec-adj-v1` | Recommended adjustments |
| `pm-active-adj-v1` | Active adjustments |

### Key Endpoints
- `/v6/credits/check/{cpUserId}` — called by Availability to get user's price quote
- `/v5/credits/quote/{cpUserId}` — price quote response format

### Segmented Pricing Rules
CPA can adjust credits cost and vendor payouts using segmented pricing rules. Segments connect to pricing rules via the Segments API (`/v1/segments` on dev/prod VPN).

### Monitoring
- Datadog: HTTP 500 errors, slow response monitors
- New Relic: APM and deployment tracking
- DB burst balance monitor

## SKU Pricing

### Purpose
DBT + Snowflake batch pipeline that calculates credit prices for ClassPass SKU items (grain bowls, smoothies) and modifiers (add-ons). New system for the non-class inventory.

### Architecture
```
SKU Source + P&I G-Sheet + Geo/Venue Metadata
    → sku_pricing_inputs (join all inputs)
    → priced_skus (core pricing logic)
    → sku_credit_amounts (CPA output) + priced_skus_snapshot + priced_skus_view (history)
```

### Exchange Rate Hierarchy (Override Priority)
1. Venue-level (most specific)
2. County-level
3. Market-level
4. Sub-area-level
5. Area-level
6. MSA-level (default, lowest priority)

### Base Credit Price Calculation
```sql
base_credit_price = LEAST(
    CEIL(negotiated_price_major_units / effective_margined_credit_price),
    COALESCE(venue_ceiling, network_ceiling)
)
```
Uses CEIL() (not ROUND) to ensure ClassPass never loses money on SKU transactions.

### Suburb Testing
- **MSA 59 Baseline**: -2 credit discount for non-excluded venues
- **Suburb Test Variants**: Deterministic assignment via `county_id % 4`, variants 1-4 with -1 or -2 credit discounts
- Minimum 1 credit enforced via `GREATEST(price - discount, 1)`

### Key Output Tables
| Table | Purpose |
|-------|---------|
| `sku_credit_amounts` | Final API-ready output for CPA |
| `priced_skus_view` | Historical audit trail (SCD2 from snapshot) |
| `sku_pricing_report` | Aggregated statistics |
| `credit_histogram` | Credit price distribution |

### Runtime
Runs every few minutes via DBT/Rednose. Pipeline completes in 1-4 minutes for all inventory.

## SmartSpot

### Purpose
For each class, SmartSpot uses tens of millions of timestamped data points to calculate how many spots a studio will fill and (with buffer) how many spots are available to ClassPass members.

### Repository
`classpass/classpass-smartspot` (Python)

### Operations
- Runs as a batch job on Parthenon (EC2 instance, shared with Platform)
- Runs from Chris's crontab via `runAffinity.sh` → `affinityScript.sh`
- Output goes to S3 (`classpass-spotmatch-state`)
- OpsGenie heartbeat: `spotmatch-affinity`

### Consumers
- **Pricing Job**: Uses fill predictions for Protection logic
- **Spot Recommendations Service**: Subscribes to SmartSpot SNS topic for spot allocation

### Data Locations
- S3: `production-platform-affinity-output` (protobuf output)
- Snowflake: `cp_pricing.affinity.affinity3`, `affinity_all`, `affinity_new`
- Analytics: `cstarace.affinity3`, etc.
- Also surfaced via Athena

### Monitoring
- Datadog: `smartspot-output-monitor` dashboard
- New Relic: APM for production and staging (backtest)

## Spot Recommendations Service

### Purpose
Receives SmartSpot fill predictions and publishes spot adjustments based on SmartSpot settings. Studio partners control settings via Partner Dashboard.

### Repository
`classpass/spot-recs-service` (Kotlin)

### Owner
Not ML Squad — separate squad owns this, but it's tightly coupled to SmartSpot.

### Architecture
```
SmartSpot S3 → SQS (spot-prediction-v3) → S3SpotRecProcessor
    → Filter/Publish → SQS (pending-spot-adjustment-v2) → SpotApplicationLogic
    → Redis (spot recommendations) → Availability
```

### Key Logic
- `SpotPredictionLogic`: Creates spot change requests, handles ssDisabled, classInThePast scenarios
- `SpotApplicationLogic`: Adjusts recommendations for max capacity, handles networkSpotsTaken, minimumSpots
- Change detection: Only publishes when spot recommendation actually changed

## Affinity Recommender

### Purpose
Generates 300 schedule recommendations per active user. Output feeds into Pricing (recommended discounts) and downstream recommendation surfaces.

### Repository
`classpass/affinity` (Java)

### Operations
- Batch job running 4x/day (every 6 hours)
- Runs from Chris's crontab on Parthenon
- Memory-intensive (dedicated New Relic memory dashboard)

### Output
Protobuf format to S3 (`production-platform-affinity-output`), also in Snowflake affinity tables.

### Migration
Being migrated to Snowflake (active project: `Affinity Migration to Snowflake`).

## Cross-System Dependencies

```
Affinity ──→ Pricing Job (recommended discounts)
SmartSpot ──→ Pricing Job (fill predictions for Protection)
SmartSpot ──→ Spot Recs (spot allocation via SNS/SQS)
Pricing Job ──→ CPA (prices via S3 → Price Loader → Redis)
PriceMatch ──→ CPA (adjustments via Airflow import)
SKU Pricing ──→ CPA (sku_credit_amounts table)
CPA ──→ Availability/Search (price quotes via API)
Segments ──→ CPA (segmented pricing rules)
```
