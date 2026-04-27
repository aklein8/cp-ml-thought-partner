# ClassPass ML Ecosystem Plugin

A comprehensive knowledge base and design partner for ClassPass ML products, technology, and ecosystem.

## What It Does

This plugin gives Claude deep, structured knowledge of the entire ClassPass ML ecosystem — 15+ models and services across pricing, recommendations, classification, fraud detection, lifecycle management, and ML platform infrastructure. It acts as a design partner for evolving the ecosystem from both product and technical perspectives.

## Skills

| Skill | Purpose |
|-------|---------|
| `ml-architecture` | Engineering systems, service dependencies, infrastructure, deployment patterns, repo map |
| `ml-pricing-engine` | Dynamic pricing, CPA, SKU pricing, SmartSpot, spot allocation, exchange rate hierarchies |
| `ml-intelligence` | LLM tagging, HAC classifier, F&B auto-tagger, review NLP, collections, tag accuracy |
| `ml-models-lifecycle` | Model catalog (Churn, RTR, LCM, Affinity, etc.), training pipelines, feature stores, monitoring |
| `ml-design-partner` | Strategic design partner — dependency graph, product roadmap, trade-off analysis, evolution planning |

## Usage

The skills trigger automatically based on your questions. Examples:

- "How does dynamic pricing work?" → triggers `ml-pricing-engine`
- "What ML models does ClassPass run?" → triggers `ml-models-lifecycle`
- "How should we evolve the classification system?" → triggers `ml-design-partner`
- "Explain the SmartSpot → Pricing → CPA data flow" → triggers `ml-architecture`
- "What is the HAC classifier and how accurate is it?" → triggers `ml-intelligence`

## Data Sources

This plugin was built from a comprehensive crawl of the ML/Data Squad Notion space (20+ pages), including:
- Service runbooks and architecture docs
- Launch gate documents (HAC, F&B, LCM, LLM Tagger)
- Active project pages (SKU Pricing, RTR Batch, P&I Migration)
- 2024 lookback metrics and impact data
- ML design principles and production checklists
- Proposals and hackathon docs

## Refreshing Content

The reference files are static snapshots. To refresh:
1. Re-crawl the ML/Data Squad Notion hub: https://www.notion.so/mindbody/ML-Data-Squad-345dda30e2318158af5fc677990a7f9b
2. Update the reference files in each skill's `references/` directory

## SharePoint TDDs (Not Yet Ingested)

These TDDs are referenced in the docs but stored in SharePoint. Upload them for deeper context:
- ML-PLAT_SupervisedModelOnboarding.docx
- TDD - Targeted User Campaigns.docx
- TDD - Hierarchical Agentic Classifier (HAC)
- TDD - F&B Autonomous Venue Tagger
- Pricing Job Migration from Data Pipeline.docx
- [DACI] N+2 Tags Expansion.docx
