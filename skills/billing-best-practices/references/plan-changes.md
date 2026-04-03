# Plan Changes

Customers change plans. Upgrades, downgrades, interval changes -- each has different timing, billing implications, and customer impact. Getting this wrong means either overcharging customers or giving away free access.

## The Core Rule

**More expensive = immediate. Less expensive = end of period.**

This is the fairest approach for both sides:
- Upgrades are immediate because the customer wants more value now and is willing to pay for it
- Downgrades wait because the customer already paid for the current period and should enjoy it

## Upgrades (More Expensive Plan)

| Aspect | Behavior |
|--------|----------|
| When applied | Immediately |
| Billing period | Resets from change date |
| Allocations | Reset to new plan values |
| Proration | Credit for unused old plan, full charge for new plan |

```typescript
import { Commet } from "@commet/node";

const commet = new Commet({ apiKey: "sk_live_..." });

await commet.subscriptions.changePlan({
  customerId: "cus_abc123",
  planId: "plan_pro_monthly",
});

// Happens immediately:
// 1. Credit calculated for unused days on old plan
// 2. New plan charged at full price (period resets)
// 3. Prorated invoice generated
// 4. Usage/credits/balance reset to new plan values
```

### What Resets on Upgrade

| Model | What Happens |
|-------|-------------|
| Metered | Usage window resets. Pending overage billed at old rate. |
| Credits | Plan credits reset to new plan value. Purchased credits preserved. |
| Balance | Balance reset to new plan's included balance. |
| Seats | New included count applies. Extra seats recalculated. |

## Downgrades (Cheaper Plan)

| Aspect | Behavior |
|--------|----------|
| When applied | At end of current period |
| During current period | Customer keeps current plan and all features |
| Credit | None -- they already paid and get to use it |
| At renewal | Switches to new plan with new price |

```typescript
await commet.subscriptions.changePlan({
  customerId: "cus_abc123",
  planId: "plan_starter_monthly",
});

// Nothing changes right now
// At next renewal: switches to Starter, charges Starter price
```

### Downgrade Warnings

#### Seats Exceeding New Limit

If the customer has 8 active seats and downgrades to a plan with 5 included seats, they can still downgrade -- but they'll pay for 3 extra seats at the new plan's rate.

Show a clear warning: "You have 8 active seats. The Starter plan includes 5. You'll be charged for 3 extra seats at $15/seat."

#### Usage Exceeding New Limit

If the customer's current usage would exceed the new plan's included amount, show the estimated overage cost at the new plan's pricing.

## Billing Interval Changes

Changing the billing interval (monthly to yearly, yearly to monthly) follows the same logic as plan changes:

| Change | Behavior | Reason |
|--------|----------|--------|
| Monthly to Yearly | Immediate | Commitment upgrade (customer gets better price) |
| Yearly to Monthly | End of period | Commitment downgrade (customer already paid for the year) |

```typescript
// Monthly to yearly -- applied immediately
await commet.subscriptions.changePlan({
  customerId: "cus_abc123",
  planId: "plan_pro_yearly", // same plan, different interval
});

// Customer gets credit for unused monthly period
// Charged full yearly price, new annual period starts
```

## Combined Changes

### Upgrade + Interval Change

If a customer upgrades their plan AND changes from monthly to yearly in the same action, it's treated as an upgrade (more expensive = immediate).

### Downgrade + Interval Change

If a customer downgrades AND changes interval, it's treated as a downgrade (cheaper = end of period).

## Intro Offers and Plan Changes

Intro offers (discounted first N months) are lost when the customer changes plans. The new plan starts at its regular price, even if the customer hadn't finished their intro offer period.

- Proration credit is based on what they actually paid (the intro price), not the list price
- One intro offer per customer lifetime -- no second chances on a new plan

## Edge Cases

### Upgrade on Day 1 of Cycle

Credit is nearly 100% of the old plan. The customer effectively pays the price difference.

### Upgrade on Last Day of Cycle

Credit is minimal. The customer paid for almost the entire old cycle before switching.

### Multiple Upgrades in Same Period

Each upgrade is calculated independently. If a customer upgrades from A to B to C in the same period, the B-to-C proration is based on B's price and the time since the B upgrade, not anything related to A.

### Upgrade Then Downgrade Request

If a customer upgrades and then immediately requests a downgrade, the downgrade is scheduled for the end of the period. No refund for the upgrade.

### Reactivation After Cancellation

A canceled customer creates a new subscription, not a plan change. No intro offers (they had a subscription before). Deprecated plans cannot be re-subscribed.

## Consumption Model Changes

Plans within the same plan group must use the same consumption model (metered, credits, or balance). Switching consumption models requires moving to a different plan group, which means canceling the current subscription and creating a new one.

## Plan Change Events

Listen for plan change events to update your application:

```typescript
app.post("/webhooks/commet", async (req, res) => {
  const event = commet.webhooks.verify(req.body, req.headers);

  switch (event.type) {
    case "subscription.plan_changed":
      await updateCustomerFeatures(
        event.data.customerId,
        event.data.newPlanId
      );
      break;

    case "subscription.plan_change_scheduled":
      // Downgrade scheduled for end of period
      await showScheduledChangeNotice(
        event.data.customerId,
        event.data.newPlanId,
        event.data.effectiveAt
      );
      break;
  }
});
```

## Related

- [Proration](./proration.md) - Detailed math for mid-cycle charges
- [Subscription Lifecycle](./subscription-lifecycle.md) - State transitions during changes
- [Invoicing](./invoicing.md) - How plan change invoices look
