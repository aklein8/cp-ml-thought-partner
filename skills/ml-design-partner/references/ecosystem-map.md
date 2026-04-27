# ClassPass ML Ecosystem Map — Design Partner Reference

## Ecosystem at a Glance

The ClassPass ML ecosystem spans 6 domains, 15+ models/services, 14+ repos, and serves both real-time and batch use cases across pricing, recommendations, classification, fraud detection, and lifecycle management.

## Domain Map

### 1. Pricing & Inventory
**Services**: Pricing Job, Credit Price API (CPA), SKU Pricing, PriceMatch
**Coupled with**: SmartSpot, Affinity, Spot Recs
**Customer impact**: Every credit price a user sees flows through this domain
**Key metrics**: 30M+ schedules priced, 171M+ Redis keys, quotes served in <600s TTL
**Active evolution**: SKU Pricing (new), Premium sunset (done), Snowflake migration (done)

### 2. Recommendations & Personalization
**Services**: Affinity Recommender, Similar Venues
**Retired**: STO/TUC (send time optimization)
**Customer impact**: 300 personalized schedule recommendations per user, feeds into pricing discounts
**Key metrics**: 4x/day batch, feeds downstream pricing and discovery
**Active evolution**: Affinity Snowflake migration, potential LLM-enhanced recommendations

### 3. Intelligence & Classification
**Services**: LLM Tagger, HAC (Hierarchical Agentic Classifier), F&B Auto Tagger, Tags Service
**Proposed**: Review NLP, Contextual/Predictive Tagging
**Customer impact**: Search quality, class discovery, new vertical onboarding, marketing collections
**Key metrics**: L1 94.5% accuracy, L2 87.5%, L3 77%, +200bps T2S conversion
**Active evolution**: HAC rollout to Movies, F&B production flip, Review Text analysis

### 4. Fraud & Trust
**Services**: RTR Real-Time (signup flow), RTR Batch (post-signup), Network Analysis, Cluster Triage
**Customer impact**: Trial abuse prevention, partner trust, FCF payout reconciliation
**Key metrics**: 80%+ deflection rate, 17 matching rules, 7 fraud type categories
**Active evolution**: Automated investigation sweep, cluster triage automation, cross-database migration

### 5. Lifecycle & Retention
**Services**: Churn Model (XGBoost), Late Cancellation Model (LCM)
**Customer impact**: Proactive save campaigns, reservation deferral for likely no-shows
**Key metrics**: $4,500+/day LCM savings, churn prediction Low/Med/High
**Active evolution**: Finance Churn Analysis, potential integration with recommendation layer

### 6. ML Platform & Infrastructure
**Services**: MLflow, Rednose (DBT), SageMaker, Parthenon (EC2)
**Customer impact**: Indirect — enables all other domains
**Key metrics**: 95% Snowflake migration, 75% P&I→DBT
**Active evolution**: P&I model migration, Coder managed workspaces

## Product Dependency Graph

```
                    ┌─────────────────┐
                    │   User Request  │
                    │  (Search/Book)  │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  Availability   │
                    │    Service      │
                    └───┬─────────┬───┘
                        │         │
               ┌────────▼───┐  ┌──▼──────────┐
               │    CPA     │  │ Tags Service │
               │  (Prices)  │  │ (Discovery)  │
               └──┬──┬──┬───┘  └──┬──────────┘
                  │  │  │         │
        ┌─────────┘  │  └──────┐  │
        │            │         │  │
   ┌────▼────┐  ┌────▼───┐ ┌──▼──▼──────┐
   │ Pricing │  │  SKU   │ │  HAC/LLM   │
   │   Job   │  │Pricing │ │  Taggers   │
   └──┬──┬───┘  └────────┘ └────────────┘
      │  │
┌─────▼┐ ┌▼──────────┐
│Affin-│ │ SmartSpot  │
│ ity  │ │(Fill Pred) │
└──────┘ └─────┬──────┘
               │
         ┌─────▼──────┐
         │  Spot Recs  │
         └─────────────┘

Side channels:
  Churn Model ──→ Lifecycle Marketing (Braze campaigns)
  LCM ──→ Reservation Service (deferral)
  RTR ──→ Signup Flow (block) + Batch (detect + reconcile)
```

## Strategic Themes & Evolution Opportunities

### Theme 1: Unification of Classification
**Current state**: Three separate classifiers (LLM Tagger, HAC, F&B Tagger) with overlapping concerns
**Opportunity**: HAC is designed to be the unified classifier. Consolidating into HAC with domain-specific prompt tuning could reduce operational complexity.
**Risks**: F&B has distinct venue-level rollup semantics; need to ensure HAC handles this
**Dependencies**: Tags Service integration, Product Ops review workflow, SEO impact assessment

### Theme 2: Real-Time Pricing Modernization
**Current state**: Pricing Job is batch (2h cycle), uses legacy patterns (pickle, single-CPU)
**Opportunity**: Parallelize pricing (segment by wellness/fitness), native serialization, real-time static pricing via CPA
**Risks**: P&I team prioritizes pricing flexibility; changes must preserve configurability
**Dependencies**: CPA capacity, Airflow orchestration, P&I stakeholder alignment

### Theme 3: Recommendation Layer Enhancement
**Current state**: Affinity generates schedule-level recs, used primarily by pricing
**Opportunity**: Feed classification metadata (collections, goal-based tags) INTO recommendations; create user-intent-aware recommendations ("I want to destress" → yoga + meditation + sound therapy)
**Dependencies**: LLM tagger collections maturity, user preference signals, recommendation model architecture

### Theme 4: Fraud Detection Automation
**Current state**: Semi-automated cluster triage with manual investigation workflow
**Opportunity**: Weekly automated investigation sweep, auto-escalation of confirmed patterns, closed-loop model improvement from investigation verdicts
**Dependencies**: Cross-database grants (CP_ML_DATA_SMARTTOOLS), Slack integration for escalation

### Theme 5: Review-Powered Intelligence
**Current state**: Proposed only — review text is underutilized
**Opportunity**: Aspect-based sentiment analysis → tag enrichment; review summaries → partner comms; quality scoring → featured review logic; moderation → safety
**Dependencies**: Review data access, LLM cost budgets, Partner team alignment

### Theme 6: ML Platform Maturity
**Current state**: Mix of legacy (Parthenon crontabs, Analytics DB) and modern (Snowflake, DBT, ECS)
**Opportunity**: Complete P&I migration to DBT, sunset Parthenon, standardize model lifecycle (training → evaluation → deployment → monitoring)
**Dependencies**: Remaining 25% P&I model migration, Parthenon decommission planning

## Key Stakeholder Map

| Stakeholder | Interest Areas |
|-------------|---------------|
| Lifecycle Marketing | Churn predictions, LLM tagging for onboarding |
| Product Ops | Tag quality, HAC review dashboard, SEO |
| P&I Team | Pricing flexibility, SmartSpot accuracy, venue settings |
| Partner Team | Partner trust, fraud detection, FCF payout reconciliation |
| Growth | New user conversion, trial experience quality |
| Catalog/Admin | Inventory tagging, new vertical onboarding |
| BI (Agnes Jiang, Adam Turkle) | Analytics, dashboards, Slack #bi-genius-bar |
| Finance | Churn analysis, pricing impact, fraud loss quantification |

## Notion Source Pages (for refreshing this reference)
- Main hub: https://www.notion.so/mindbody/ML-Data-Squad-345dda30e2318158af5fc677990a7f9b
- Active Projects: https://www.notion.so/345dda30e231818b9850d3991061ac55
- The World According to ML/Data: https://www.notion.so/345dda30e23181318159cf12020149d7
- 2024 Lookback: https://www.notion.so/345dda30e23181d6bd9ee2d96c446954
- Proposals: https://www.notion.so/345dda30e23181cfad9ce0c1bb359361
