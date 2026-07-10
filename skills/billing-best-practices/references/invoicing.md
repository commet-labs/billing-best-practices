# Invoicing

Invoices are the financial record of every charge. Understanding when they're generated, what they contain, and how the numbers are calculated is essential for building a billing system that your customers (and your accountant) can trust.

## Invoice Types

| Invoice | When Generated | Contains |
|---------|---------------|----------|
| Recurring | Every billing cycle | Plan base + usage overage + seats + addons |
| Overage | Between billing cycles (quarterly/yearly only) | Usage overage only |
| Plan change | Customer upgrades mid-cycle | Credit for old plan + charge for new plan |
| Credit purchase | Customer buys a credit pack | Credit pack amount |
| Balance top-up | Customer adds funds to balance | Top-up amount |
| Adjustment | Manual correction from dashboard | One-off charge or credit |

## Recurring Invoice

The standard invoice generated at each billing cycle. This is what most customers see most of the time.

```
Monthly invoice for Pro plan at $99/mo:

  Plan base (Pro - Monthly):                        $99.00
  Extra usage (2,000 API calls x $0.01):             $20.00
  Extra seats (2 x $25):                             $50.00
  Addon (Priority Support):                          $29.00
  ─────────────────────────────────────────────────────────
  Subtotal:                                         $198.00
  Promo code (SAVE20 - 20% off):                    -$39.60
  ─────────────────────────────────────────────────────────
  Total:                                            $158.40
```

## Advance vs True-Up Charges

Different items are charged at different points in the cycle:

| Charge Type | When Billed | Examples |
|-------------|------------|---------|
| Advance | Period start (for the upcoming period) | Plan base, boolean features, addon base |
| True-up | Period end (for the previous period) | Metered overage, balance overage |
| Hybrid | Both advance and true-up | Seats (advance for included, true-up for extras) |

This means a recurring invoice can contain charges for two different periods:
- Plan base for the **next** period (advance)
- Usage overage from the **previous** period (true-up)

## Invoice Line Types

| Line Type | Description |
|-----------|-------------|
| `plan_base` | Base plan price charged in advance |
| `feature_overage` | Usage beyond included amount (metered features) |
| `feature_seats` | Seat charges (advance or true-up) |
| `addon_base` | Addon recurring charge |
| `balance_overage` | Overage when balance model allows going negative |
| `discount` | Intro offer discount |
| `promo_code_discount` | Promo code discount |
| `credit` | Customer credit applied to reduce total |

## Billing Cycles

### Monthly

Every billing cycle is a full invoice with all charges. Simple.

### Quarterly and Yearly

Quotas (usage, credits, balance) reset **monthly**, even though the plan is billed quarterly or yearly. This creates a split:

- **Billing month** (every 3 or 12 months): Full invoice with plan base, seats, usage, addons
- **Non-billing month**: Only overage is billed (if any)

```
Yearly plan, billing month = January:

Jan (billing month): Plan base + seats + usage + addons = full invoice
Feb (non-billing):   Usage overage only (if any)
Mar (non-billing):   Usage overage only (if any)
...
Dec (non-billing):   Usage overage only (if any)
Jan (billing month): Full invoice again
```

### Non-Billing Month Behavior by Model

| Model | Non-Billing Month |
|-------|------------------|
| Metered | Mini-invoice with overage if any, otherwise nothing |
| Credits | Reset only, no invoice (credits block when exhausted) |
| Balance (block on exhaustion) | Reset only, no invoice |
| Balance (allow overage) | Mini-invoice if overage > $0, otherwise reset only |

Seats are only billed on billing months (advance for the full interval).

## Consumption Models and Invoicing

### Metered

```typescript
import { Commet } from "@commet/node";

const commet = new Commet({ apiKey: process.env.COMMET_API_KEY! });

// Track usage throughout the period
await commet.usage.track({
  customerId: "cus_abc123",
  feature: "api_calls",
  value: 1,
});

// At billing cycle: usage beyond included amount is charged
// Invoice line: feature_overage, quantity: 2000, unitAmount: $0.01
```

Invoice includes: plan base (advance) + overage (true-up for previous period)

### Credits

```typescript
// Customer uses credits throughout the period
// When credits reach 0: access blocked immediately
// Customer can purchase credit packs to continue

await commet.subscriptions.purchaseCredits({
  id: "sub_xxx",
  creditPackId: "cpk_xxx",
});
```

Invoice includes: plan base (advance). No overage -- credits block when exhausted. Credit pack purchases generate separate invoices.

### Balance

```typescript
// Usage deducts from balance in real time
// Behavior depends on blockOnExhaustion setting:
// - true: access blocked when balance hits 0
// - false: balance can go negative, overage charged at period end
```

Invoice includes: plan base (advance) + balance overage (true-up, if balance model allows going negative)

## Rate Scale vs Settlement Scale

Two precision levels coexist in invoicing:

| Field | Scale | Precision | Example |
|-------|-------|-----------|---------|
| Line item `unitAmount` | Rate (10000 = $1) | Sub-cent pricing | `150` = $0.015/unit |
| Invoice `total` | Settlement (100 = $1) | Standard cents | `9900` = $99.00 |
| Plan base `amount` | Settlement (100 = $1) | Standard cents | `9900` = $99.00 |

**Why two scales?** Rate scale enables sub-cent pricing for metered billing (e.g., $0.001 per API call = `10` in rate scale). Settlement scale is what payment processors understand.

## Plan Change Invoice

When a customer upgrades mid-cycle, a special invoice is generated:

```
Plan change from Starter to Pro on January 15:

  Credit for unused Starter (15 days):     -$14.50
  Pro plan (full cycle, Jan 15 - Feb 15):   $99.00
  Pending overage (5K calls x $0.02):      $100.00
  ─────────────────────────────────────────────────
  Total:                                   $184.50
```

## Currency on Invoices

All line amounts are in the subscription's currency. If the customer pays in BRL, every line item, subtotal, and total is in BRL. The billing system tracks both the presentment amount (what the customer paid) and the USD settlement amount (what Stripe deposited) separately.

## Related

- [Proration](./proration.md) - How plan change invoice amounts are calculated
- [Subscription Lifecycle](./subscription-lifecycle.md) - When invoices are generated in each state
- [Multi-Currency](./multi-currency.md) - Currency handling on invoices
- [Billing Checklist](./billing-checklist.md) - Invoice testing before launch
