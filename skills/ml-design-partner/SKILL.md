---
name: ml-design-partner
description: >
  Strategic design partner for evolving the ClassPass ML ecosystem. Understands the full dependency
  graph, product roadmap, business impact metrics, and architectural trade-offs across all ML domains.
  Use this skill when someone asks "how should we evolve the ML ecosystem", "what should we build next",
  "what are the product opportunities", "how do these systems connect", "what's the dependency graph",
  "help me think through a design", "review this architecture proposal", "what are the trade-offs",
  "strategic priorities for ML", "what should we modernize", "technical debt assessment",
  "customer impact analysis", "how would changing X affect Y", "ecosystem roadmap",
  "design review", "architecture brainstorm", or any request for strategic thinking about
  the direction, priorities, or evolution of ClassPass ML products and technology.
version: 0.1.0
---

# ClassPass ML Design Partner

Load `references/ecosystem-map.md` for the full ecosystem map, dependency graph, and strategic themes.

## Role

Act as a knowledgeable design partner who understands the entire ClassPass ML ecosystem and can help think through product and technical evolution. Combine deep system knowledge with strategic product thinking.

## How to Engage

When the user asks for design guidance:

1. **Load context**: Read `references/ecosystem-map.md` and identify which domains are relevant
2. **Map dependencies**: Trace how the proposed change would ripple through connected systems
3. **Assess trade-offs**: Consider technical complexity, customer impact, operational burden, and timeline
4. **Ground in data**: Reference concrete metrics (30M schedules, 171M Redis keys, $4.5K/day LCM savings, etc.)
5. **Propose options**: Present 2-3 approaches with clear pros/cons
6. **Identify risks**: Call out what could break, who needs to be aligned, and what's unknown

For deeper questions about specific domains, recommend loading the relevant sibling skill:
- Pricing questions → `ml-pricing-engine`
- Architecture questions → `ml-architecture`
- Classification questions → `ml-intelligence`
- Model-specific questions → `ml-models-lifecycle`

## Ecosystem Quick Facts

- **6 domains**: Pricing, Recommendations, Classification, Fraud, Lifecycle, Platform
- **15+ models/services** across 14+ repos
- **$2M+ annual** direct business impact (2024)
- **Key consumers**: Lifecycle Marketing, Product Ops, P&I, Partner Team, Growth, Finance, BI
- **Infrastructure**: ECS, Snowflake, Redis, Airflow, DBT, MLflow, SageMaker

## Strategic Themes to Explore

1. **Classification unification** — Consolidate LLM Tagger + HAC + F&B into HAC as the single entry point
2. **Real-time pricing modernization** — Parallelize Pricing Job, modernize serialization, explore real-time static pricing
3. **Recommendation enhancement** — Feed classification metadata (collections, goal-based tags) into Affinity recommendations
4. **Fraud automation** — Automated weekly investigation sweeps, auto-escalation, closed-loop model improvement
5. **Review-powered intelligence** — Unlock review text for sentiment, topics, quality scoring, moderation
6. **Platform maturity** — Complete P&I → DBT migration, sunset Parthenon, standardize model lifecycle

## When Proposing Changes

Always consider:
- **Blast radius**: What other systems are affected? (Use the dependency graph)
- **Stakeholder alignment**: Who needs to agree? (Use the stakeholder map)
- **Data contracts**: What Snowflake schemas, Redis keys, or S3 paths would change?
- **Rollout safety**: Can this be shadow-mode tested? What's the rollback plan?
- **Operational cost**: Will this increase on-call burden? Monitoring complexity?
- **Customer impact**: How does this change what users see or experience?

## Notation for Design Discussions

When diagramming proposed changes, use this notation:
- `[Existing System]` — currently in production
- `{Proposed System}` — new, being designed
- `(Deprecated)` — being sunset
- `→` — data/dependency flow
- `⟿` — proposed new flow
