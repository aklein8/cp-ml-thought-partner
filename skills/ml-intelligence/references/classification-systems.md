# ClassPass Intelligence & Classification Systems

## Overview

The classification layer provides automated metadata generation for all ClassPass inventory. It has evolved from simple keyword-based tagging to a multi-model LLM-powered system that covers fitness, wellness, food & beverage, and new verticals (movies, events, etc.).

## LLM Tagging Service

### Purpose
Automatically tags classes based on class names, descriptions, and user review data using LLMs. Enhances class categorization for recommendations and marketing.

### Launch
September 2024 (Phase 1: LFC Beginner Friendly in New York)

### Technical Implementation
- **Batch process**: Reads from Snowflake tables, writes to `CP_MLDATA.RECOMMENDER.LLM_TAGGING_OUTPUT`
- **Not integrated** directly with existing tagging services or tables
- **Not available** as self-service API (quality and cost control)
- **LLM**: OpenAI-based classification

### Data Flow
**Input**: Target tag, MSA(s), class descriptions, user review data
**Output** (Snowflake table fields):
- MARKET_ID, MARKET, VENUE_ID, CONTENT_NAME, CLASS_ID, VENUE_NAME
- CLASS_DESCRIPTION, CONTEXT, INPUT_TAGS, MODEL
- OUTPUT_TAGS, CHAT_HISTORY, REASON, UPDATED_AT

### Success Metrics
- **Primary**: Trial to Subscription (T2S) Conversion Rate per LFC test by MSA
- **Secondary**: Class conversion rate increase for newly tagged groups
- **Cost**: Tracking and minimizing LLM call costs

### Results
The review-enhanced model vs. older description-only model in NYC Beginner Friendly:
- **New model**: 1,118 classes, 535 unique venues
- **Old model**: 441 classes, 280 unique venues
- **Overlap**: 331 classes, 225 venues
- 2x+ expanded inventory of "beginner friendly" classes

### Key Innovation: Collections
The service enables "collections" — new class groupings like "restorative", "motivational", "strength building" that cross genre boundaries. These were not previously feasible manually at scale and unlock metadata potential for goal-based recommendation models.

## Hierarchical Agentic Classifier (HAC)

### Purpose
Unified tagging model for ALL inventory (excluding F&B, which has its own tagger). Initial rollout limited to new verticals (Movies first), with tags generated but held for existing domains (Wellness, Fitness) until human-in-the-loop is ready.

### Architecture
```
Class/Venue Update → SQS Event Queue → HAC Worker
    → Preprocessing Engine (clean/normalize text, pull taxonomy from Snowflake via Fivetran)
    → LLM Integration (OpenAI classifier, predicts L1/L2/L3 tags)
    → Postgres (predictions + tag mappings) → Snowflake replication
```

### Key Design Decisions
- Tagging at **class/item level** (not venue level)
- Uses both **class and venue metadata** (name + description) as input
- **All tags stored** even for non-enabled domains — future rollouts apply via DB lookup
- **Change detection**: Skips processing when class/venue info hasn't changed
- **Venue type filtering**: Excludes F&B (handled by F&B Auto Tagger)

### Rollout Phases
1. **Development & Testing**: Pipeline, SQS, preprocessing, LLM, DB storage
2. **Shadow Mode**: Dry run without production tagging. Accuracy: L1 94.5%, L2 87.5%, L3 77%
3. **Monitoring & Evaluation**: Unit tests, integration tests, Datadog dashboards, New Relic logging
4. **Production Rollout**: Enable for Movies only (first new vertical)
5. **Post-Deployment**: Hex review dashboard for Product Ops, runbook for ML team

### Monitoring
- Datadog: `recommender-hierarchical-tagging-worker` service APM
- New Relic: Worker performance logging
- Snowflake: `CP_PROD.RECOMMENDER_PUBLIC.AGENTIC_CLASSFIER_PREDICTIONS`
- Hex: Review dashboard for prediction auditing

## F&B Auto Tagger

### Purpose
Automatically tags Food & Beverage items as they come live. Separate from HAC because F&B was the first vertical with auto-tagging needs.

### Architecture
Same pattern as HAC:
```
Venue Event → SQS → F&B Tagging Worker → OpenAI LLM → Postgres → Snowflake
```

### Key Features
- Class/item level tagging (rolls up to venue level)
- SQS worker listens to venue events queue
- REST API controllers for manual tagging and batch operations
- Change detection: skips when name/description unchanged
- Venue type filtering: only F&B venues
- Prediction logging for analysis
- MLflow prompt management (`fb-item-tagging-prompt`)

### Accuracy
~>90% accurate on tagging (some edge cases)

### Rollout Status
- Phases 1-3 complete (development, shadow mode, monitoring)
- Phase 4 pending: flip switch to tag F&B items in production
- Phase 5 pending: review dashboard for tagging team

### Open Questions
- SEO implications of class-level tagging with venue rollup
- Error handling: MLflow prompt updates fix future errors; past error correction needs override/retag endpoints
- Batch operations (backfill endpoint) not in scope for initial phase

## LLM-Based Inventory Metadata Generation (Product Vision)

### Long-Term Vision
Extend LLM tagging into a comprehensive metadata generation system for ALL ClassPass inventory including F&B expansion.

### Core Features (Proposed)
1. **Automated Metadata Generation**: Auto-tag from class descriptions, reviews, menus
2. **Tag Validation and Correction**: Historical data analysis, inconsistency flagging (e.g., both "Advanced" and "Beginner Friendly")
3. **Contextual Tagging**: Dynamic tags based on user preferences, time, location, behavior (e.g., "Morning Yoga")
4. **Predictive Tagging for New Inventory**: Predict tags from descriptions before manual input
5. **Tag Interaction Learning**: Learn tag combinations from user behavior (Yoga users also search Meditation)

### Stakeholders
- D: Ryan, A: Megan
- C: ML, Growth, Catalog, Admin, Product Ops, LFC, Partners
- I: P&I, Product

### Risks
- Tag interaction complexity across categories
- User confusion from tag overload (mitigate by prioritizing relevant tags)
- Backward compatibility with hierarchical tagging system

## Review Text Analysis (Proposed)

### Capabilities Matrix
| Component | Method | Application | Business Insight |
|-----------|--------|-------------|-----------------|
| Sentiment & Rating Consistency | Sentiment Analysis + Rating Check | Identify tone, disparity between sentiment and rating | Track satisfaction, validate review reliability |
| Topic & Keyword Extraction | Topic Modeling + Keyword Extraction | Extract main topics, phrases, keywords | Highlight popular classes/trainers/studios, recurring themes |
| Aspect-Specific Sentiment | Aspect-Based Sentiment Analysis | Sentiment on "class quality", "location", "trainer" | Discover strengths/weaknesses by class or location |
| Review Quality Scoring | Composite Scoring (length, detail) | Evaluate richness, detail, credibility | Prioritize high-quality reviews for booking decisions |
| Content Moderation | Text Classification | Detect inappropriate/sensitive content | Ensure safe, inclusive platform |
| Actionable Insights | Review Summarization | Concise partner feedback, user-facing review summaries | Surface all review types to partners; help users make informed decisions |

## Tags Service Integration

The Tags service (`classpass/tags`) is the central system that stores and serves tags:
- Production service for real-time tag queries
- Connected to Snowflake via `cp_prod.tags_public.view_content_tag`
- BI derived: `cp_bi_derived.datapipeline.class_tags`
- HAC and F&B tagger write predictions to Postgres which replicates to Snowflake; tags service reads from this

## Cross-System Dependencies

```
LLM Tagger ──→ Snowflake (LLM_TAGGING_OUTPUT) ──→ LFC Marketing Tests
HAC Worker ──→ Postgres ──→ Snowflake (AGENTIC_CLASSIFIER_PREDICTIONS)
F&B Tagger ──→ Postgres ──→ Snowflake
Tags Service ──→ Search/Discovery ──→ User Experience
Review NLP (proposed) ──→ Tags enrichment + Partner comms + Venue details
All Classifiers ──→ MLflow (prompt management, experiment tracking)
```
