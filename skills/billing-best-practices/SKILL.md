---
name: billing-best-practices
description: Use when building SaaS billing features, handling failed payments, implementing proration, supporting multiple currencies, managing subscription lifecycle, setting up tax compliance, or designing invoicing flows. Contains patterns and best practices for production billing systems.
license: MIT
metadata:
  author: commet
  version: "1.0.0"
  homepage: https://commet.co
  source: https://github.com/commet-labs/billing-best-practices
references:
  - references/subscription-lifecycle.md
  - references/failed-payments.md
  - references/proration.md
  - references/multi-currency.md
  - references/tax-compliance.md
  - references/plan-changes.md
  - references/invoicing.md
  - references/billing-checklist.md
---

# Billing Best Practices

Guidance for building production SaaS billing systems that handle real-world complexity: failed payments, mid-cycle changes, multiple currencies, tax compliance, and everything in between.

## Architecture Overview

```
[Customer] --> [Checkout] --> [Subscription Created]
                                      |
                    +-----------------+-----------------+
                    |                 |                 |
              [Active]          [Trialing]       [Past Due]
                    |                 |                 |
              [Billing Cycle]   [Trial Ends]     [Retry Logic]
                    |                 |                 |
              [Invoice]         [First Charge]   [Recovered / Canceled]
                    |                 |
              [Payment]         [Active]
                    |
              [Next Cycle]
```

## Quick Reference

| Need to... | See |
|------------|-----|
| Understand subscription states and transitions | [Subscription Lifecycle](./references/subscription-lifecycle.md) |
| Handle failed payments and reduce churn | [Failed Payments](./references/failed-payments.md) |
| Calculate mid-cycle upgrade/downgrade charges | [Proration](./references/proration.md) |
| Support multiple currencies and regional pricing | [Multi-Currency](./references/multi-currency.md) |
| Set up tax collection and compliance | [Tax Compliance](./references/tax-compliance.md) |
| Implement plan upgrades and downgrades | [Plan Changes](./references/plan-changes.md) |
| Design invoice structure and billing cycles | [Invoicing](./references/invoicing.md) |
| Prepare for launch with a billing checklist | [Billing Checklist](./references/billing-checklist.md) |

## Start Here

**Building billing from scratch?**
Start with [Subscription Lifecycle](./references/subscription-lifecycle.md) to understand how subscriptions move through states, then [Invoicing](./references/invoicing.md) to design your billing cycle. Run through the [Billing Checklist](./references/billing-checklist.md) before going live.

**Losing customers to failed payments?**
Go directly to [Failed Payments](./references/failed-payments.md). Involuntary churn from payment failures is the most preventable source of revenue loss.

**Adding plan upgrades/downgrades?**
Follow this path: [Plan Changes](./references/plan-changes.md) (when and how changes apply) then [Proration](./references/proration.md) (the math behind mid-cycle charges).

**Going international?**
Start with [Multi-Currency](./references/multi-currency.md) (currency detection, regional pricing) then [Tax Compliance](./references/tax-compliance.md) (automated tax collection, MoR model).
