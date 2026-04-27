---
name: ml-intelligence
description: >
  Deep knowledge of ClassPass classification and metadata generation systems including LLM-based
  tagging, the Hierarchical Agentic Classifier (HAC), F&B auto-tagger, and review text analysis.
  Use this skill when someone asks "how does tagging work", "what is HAC", "how are classes tagged",
  "explain the LLM tagger", "what is the F&B auto tagger", "how do collections work",
  "what is contextual tagging", "explain the classification pipeline", "review NLP",
  "how does inventory metadata generation work", "tag accuracy", "what verticals are tagged",
  or any question about how ClassPass automatically categorizes, tags, or generates metadata
  for its inventory of classes, venues, and items.
version: 0.1.0
---

# ClassPass Intelligence & Classification

Load `references/classification-systems.md` for complete classification documentation.

## Quick Orientation

The classification layer has evolved from manual tagging to an LLM-powered multi-model system:

1. **LLM Tagger** (2024) — Batch process tagging classes using descriptions + review data. Writes to Snowflake. Powers marketing collections and onboarding experiments.
2. **HAC** (2025) — Unified classifier for ALL inventory (except F&B). Uses OpenAI, SQS-driven, stores all predictions (even for non-enabled domains). Currently live for Movies.
3. **F&B Auto Tagger** (2025) — Dedicated F&B item tagger. Same architecture as HAC but separate because F&B had early needs. ~90%+ accuracy.
4. **Review NLP** (Proposed) — Sentiment analysis, topic extraction, aspect-based analysis, quality scoring, moderation, summarization.

## When Answering Classification Questions

1. Read `references/classification-systems.md` for the full documentation
2. Identify which classifier handles the inventory type (HAC for non-F&B, F&B Tagger for food/bev, LLM Tagger for marketing collections)
3. Understand the tag hierarchy: L1, L2, L3 (HAC predicts all three levels)
4. Note the data flow: SQS event → Worker → Preprocessing → LLM → Postgres → Snowflake
5. Distinguish between tag storage (all tags stored) and tag application (only enabled domains)

## Key Concepts

- **Collections**: Cross-genre class groupings ("restorative", "strength building") enabled by LLM tagging. Not feasible manually at scale.
- **Shadow mode**: Classifiers run and log predictions without applying to production data
- **Change detection**: Workers skip processing when class/venue info hasn't changed
- **MLflow prompt management**: F&B tagger uses MLflow for prompt versioning
- **HAC accuracy**: L1 94.5%, L2 87.5%, L3 77%
- **Hex review dashboards**: Product Ops can audit HAC and F&B predictions

## Evolution Path

```
LLM Tagger (batch, marketing)
    ↓ learning
F&B Auto Tagger (event-driven, F&B only)
    ↓ generalization
HAC (event-driven, all inventory)
    ↓ future
Unified Classification + Review NLP + Contextual/Predictive Tagging
```
