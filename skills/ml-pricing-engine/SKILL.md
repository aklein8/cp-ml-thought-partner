---
name: ml-pricing-engine
description: >
  Deep knowledge of ClassPass pricing and inventory systems including dynamic pricing,
  credit price calculations, SmartSpot fill predictions, and spot allocation. Use this skill
  when someone asks "how does pricing work", "what is CPA", "how are credit prices calculated",
  "explain dynamic pricing", "what is SmartSpot", "how does protection work", "what is SKU pricing",
  "how does PriceMatch work", "explain the pricing pipeline", "what is the exchange rate hierarchy",
  "how do spot recommendations work", "pricing architecture", or any question about how ClassPass
  determines what users pay in credits, how partner payouts are calculated, or how inventory
  spots are allocated.
version: 0.1.0
---

# ClassPass Pricing Engine

Load `references/pricing-systems.md` for complete pricing documentation.

## Quick Orientation

Pricing is the most interconnected domain in the ML ecosystem. The core question it answers: **how many credits should this class/SKU cost for this user?**

The answer flows through multiple systems:
1. **Pricing Job** (batch, every 2h) — calculates base prices for 30M+ schedules using dynamic pricing
2. **SKU Pricing** (batch, DBT) — calculates prices for food/beverage/non-class items
3. **CPA** (real-time, Kotlin) — serves price quotes to users via Redis, applies per-user pricing rules
4. **SmartSpot** (batch, 4x/day) — predicts studio fill rates, feeds into Protection logic
5. **Spot Recs** (event-driven) — translates SmartSpot predictions into spot allocation adjustments

## When Answering Pricing Questions

1. Read `references/pricing-systems.md` for the full system documentation
2. Determine which system the question targets (schedule pricing vs SKU pricing vs spot allocation)
3. Trace the price calculation: data sources → pipeline → Redis → CPA → user
4. Distinguish between batch-calculated prices (Pricing Job, SKU) and real-time adjustments (pricing rules, PriceMatch)
5. Note the exchange rate hierarchy (venue > county > market > sub-area > area > MSA)

## Key Pricing Concepts

- **Protection**: As a class fills up, minimum price increases toward retail rate
- **Dynamic pricing**: Prices adjust based on SmartSpot fill predictions and demand signals
- **Segmented pricing rules**: CPA can apply per-user price adjustments via the Segments service
- **Network vs Premium**: Premium pricing deprecated Dec 2024; everything is network now
- **CEIL() rounding**: Credit prices always round UP to protect margins
- **Suburb testing**: A/B tests with -1 or -2 credit discounts in suburban areas

## Critical Data Flows

```
SmartSpot → Pricing Job → S3 → CPA Price Loader → Redis → Availability → User
Affinity → Pricing Job (recommended discounts)
PriceMatch → CPA (hourly adjustment import)
SKU Pricing (DBT) → sku_credit_amounts → CPA
```

## Open Improvement Areas
- Pricing Job parallelization (currently single-CPU on 12+ CPU machines)
- Serialization modernization (pickle → cPickle, json → ultrajson)
- Real-time static pricing via CPA (bypass batch for static schedules)
