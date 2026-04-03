# Proration

When a customer changes plans mid-cycle, you need to calculate what they owe. The principle is simple: credit them for the unused time on the old plan, charge them for the new plan. The details are where it gets interesting.

## The Core Formula

```
Credit = Old price x (days remaining / days in cycle)
New charge = New plan price (full cycle, period resets)
Customer pays = New charge - Credit
```

Upgrades reset the billing period. The customer starts a fresh full cycle on the new plan and gets credit for the unused portion of the old one.

## Simple Upgrade

```
Customer on Starter at $29/mo, paid January 1
Upgrades to Pro at $99/mo on January 15 (15 days remaining of 30)

Credit: $29 x (15/30) = $14.50
Charge: $99 (full price, new period: Jan 15 - Feb 15)
They pay: $84.50
```

```typescript
import { Commet } from "@commet/node";

const commet = new Commet({ apiKey: "sk_live_..." });

// Upgrades are applied immediately with automatic proration
await commet.subscriptions.changePlan({
  customerId: "cus_abc123",
  planId: "plan_pro_monthly",
});

// Prorated invoice is generated automatically
// Customer charged $84.50 (new price minus unused credit)
```

## Upgrade with Extra Seats

When the customer has extra seats beyond the plan's included count, those are prorated too.

```
Current plan: Pro $99/mo, 5 included seats, $25/extra
Customer has 8 seats (3 extra = $75/mo in seat charges)

Upgrades to: Business $299/mo, 10 included seats, $20/extra
On January 15 (15 days remaining of 30)

Plan credit:  $99 x (15/30)  = $49.50
Seats credit: $75 x (15/30)  = $37.50
Total credit:                   $87.00

New plan charge: $299 (full, period resets)
New seat charge: $0 (8 seats within 10 included)

They pay: $299 - $87.00 = $212.00
```

The customer's 8 seats are now covered by the Business plan's 10 included seats, so they stop paying for extra seats.

## Upgrade with Pending Overage

If the customer has usage overage on a metered plan at the time of upgrade, that overage is billed at the old plan's rate.

```
Current plan: Starter $29/mo, 10K API calls included, $0.02/extra
Customer has used 15K calls (5K overage)

Upgrades to: Pro $99/mo, 100K calls, $0.01/extra

Plan credit:      $29 x (15/30) = $14.50
Pending overage:  5K x $0.02    = $100.00 (old plan rate)
New charge:       $99 (full, period resets)

They pay: $99 - $14.50 + $100.00 = $184.50
Usage resets to 0 on the new plan.
```

## Credits Model Upgrade

For plans using the credits consumption model, proration is still time-based -- not based on credits consumed.

```
Current plan: Starter $20/mo, 500 credits included (paid Jan 1)
Customer has: planCredits = 60, purchasedCredits = 30

Upgrades to: Pro $40/mo, 1000 credits
On January 15

Credit: $20 x (15/30) = $10.00 (time-based, regardless of credits used)
Charge: $40 (full, period resets)
They pay: $30.00

After upgrade:
  planCredits: 1000 (reset to new plan value)
  purchasedCredits: 30 (always preserved)
```

This is the simplest and fairest approach: you paid for N days, used M, you get refunded for the rest proportionally.

## Quarterly and Yearly Plans

The same formula applies -- only the cycle length changes.

```
Customer on Plan A at $300/quarter (Jan 1 - Apr 1)
Upgrades to Plan B at $600/quarter on February 15 (45 days remaining of 90)

Credit: $300 x (45/90) = $150.00
Charge: $600 (full quarter, period resets)
They pay: $450.00

Next renewal: May 15
```

## Why Downgrades Aren't Prorated

Downgrades take effect at the end of the current period. The customer already paid for the cycle and keeps their current plan until it expires. Since there's no mid-cycle switch, there's nothing to prorate.

```typescript
// Downgrades are scheduled, not immediate
await commet.subscriptions.changePlan({
  customerId: "cus_abc123",
  planId: "plan_starter_monthly",
});

// Customer keeps Pro plan until current period ends
// At renewal, switches to Starter at Starter price
```

## Why Free to Paid Isn't Prorated

When a customer moves from free to paid, there's no credit to give -- the free plan costs $0. They pay the full price of the new plan from day one.

## Intro Offer Proration

If the customer is on an intro offer (discounted first N months), the credit calculation uses the **effective price** -- what they actually paid, not the list price.

```
Customer on Pro at intro price of $49/mo (list: $99)
Upgrades to Business $299/mo on day 15

Credit: $49 x (15/30) = $24.50 (based on what they paid)
Charge: $299 (full price, intro offer lost on plan change)
They pay: $274.50
```

Intro offers are lost when changing plans. The new plan starts at its regular price.

## Common Gotchas

1. **Credit is time-based, not usage-based.** Even if a customer used 100% of their credits in the first day, the credit is calculated on remaining days. This is simpler and fairer.

2. **Period resets on upgrade.** The new billing period starts from the upgrade date, not the original billing day.

3. **Purchased credits survive everything.** Plan credits reset on upgrade, but purchased credits are never touched.

4. **Multiple upgrades in the same period** are each calculated independently. No accumulation from previous changes.

5. **Upgrade on day 1** gives nearly 100% credit for the old plan -- the customer pays roughly the price difference.

6. **Upgrade on the last day** gives minimal credit. The customer paid for almost the full old cycle.

## Related

- [Plan Changes](./plan-changes.md) - When changes are immediate vs end-of-period
- [Invoicing](./invoicing.md) - How proration appears on invoices
- [Subscription Lifecycle](./subscription-lifecycle.md) - State transitions during plan changes
