# Subscription Lifecycle

Every subscription moves through a set of states. Understanding these transitions is the foundation of billing -- get them wrong and you'll either give away free access or lock out paying customers.

## States

```
         ┌─────────┐
         │  draft   │
         └────┬─────┘
              │ checkout initiated
              v
     ┌─────────────────┐
     │ pending_payment  │
     └───────┬─────────┘
             │
      ┌──────┴──────┐
      │             │
      v             v
┌──────────┐  ┌──────────┐
│ trialing │  │  active   │◄─── payment succeeds
└────┬─────┘  └─────┬─────┘
     │ trial ends    │
     │ + payment ok  │
     v               │
   active ◄──────────┘
     │
     ├── payment fails ──► past_due ──► canceled (retries exhausted)
     ├── customer cancels ──► canceled
     └── paused ──► resumed ──► active
```

| State | Access | Billing | Transitions To |
|-------|--------|---------|---------------|
| `draft` | No | None | `pending_payment` |
| `pending_payment` | No | Awaiting payment | `trialing`, `active` |
| `trialing` | Yes | No charge (usage accumulates) | `active`, `canceled` |
| `active` | Yes | Normal billing | `past_due`, `paused`, `canceled` |
| `past_due` | Yes (grace period) | Retrying payment | `active`, `canceled` |
| `paused` | No | Skipped | `active` |
| `canceled` | No | None | New subscription only |

## One Active Subscription Per Customer

A customer can only have one active subscription at a time. States that block new subscription creation: `draft`, `pending_payment`, `trialing`, `active`, `paused`, `past_due`. Only `canceled` and `expired` allow a new subscription.

## Creation Flows

### Paid Plan (No Trial)

```typescript
import { Commet } from "@commet/node";

const commet = new Commet({ apiKey: "sk_live_..." });

// Create a subscription -- customer is redirected to checkout
const subscription = await commet.subscriptions.create({
  customerId: "cus_abc123",
  planId: "plan_pro_monthly",
});

// Status: pending_payment
// Customer completes checkout --> status: active
// First invoice generated immediately
```

### Trial Period

Trials require a payment method upfront (collected via setup checkout) but don't charge until the trial ends.

```typescript
const subscription = await commet.subscriptions.create({
  customerId: "cus_abc123",
  planId: "plan_pro_monthly", // plan has trialDays: 14
});

// Status: pending_payment
// Customer completes setup checkout (card saved, no charge)
// Status: trialing, trialEndsAt set to 14 days from now (midnight UTC)
// Trial expires --> billing engine charges first invoice --> status: active
```

During the trial: full access, no charge, usage accumulates. If the first payment fails when the trial ends, the subscription moves to `past_due` and follows the normal retry flow.

### Free Plan

```typescript
const subscription = await commet.subscriptions.create({
  customerId: "cus_abc123",
  planId: "plan_free",
});

// Status: active immediately
// No billing interval, no invoice, no payment method required
```

## Billing Period

Every paid subscription tracks its billing cycle:

| Field | Purpose |
|-------|---------|
| `billingInterval` | `monthly`, `quarterly`, `yearly` (null for free plans) |
| `billingDayOfMonth` | Day the cycle renews (1-28) |
| `currentPeriodStart` | Start of current billing period |
| `currentPeriodEnd` | End of current billing period |
| `lastBilledAt` | When the last invoice was generated |
| `currency` | Locked after first payment |

## Checking Subscription State

### Webhooks (Recommended)

Listen for state change events and update your app in real time. This is the most reliable approach -- you never miss a transition.

```typescript
// In your webhook handler
app.post("/webhooks/commet", async (req, res) => {
  const event = commet.webhooks.verify(req.body, req.headers);

  switch (event.type) {
    case "subscription.activated":
      await grantAccess(event.data.customerId);
      break;

    case "subscription.past_due":
      await showPaymentBanner(event.data.customerId);
      break;

    case "subscription.canceled":
      await revokeAccess(event.data.customerId);
      break;

    case "subscription.trial_ending":
      await sendTrialEndingEmail(event.data.customerId);
      break;
  }
});
```

### Polling (Fallback)

If you can't use webhooks, check subscription state on each request. Cache the result to avoid excessive API calls.

```typescript
async function checkAccess(customerId: string): Promise<boolean> {
  const subscription = await commet.subscriptions.getByCustomer(customerId);

  if (!subscription) return false;

  const hasAccess = ["trialing", "active", "past_due"].includes(
    subscription.status
  );

  return hasAccess;
}
```

**Why `past_due` grants access:** The customer is in a grace period while payment is retried. Cutting them off immediately increases churn -- most failed payments are recovered automatically.

## Pause and Resume

Pausing stops billing and access. The customer's subscription is frozen -- no usage accumulates, no invoices are generated.

```typescript
await commet.subscriptions.pause({
  customerId: "cus_abc123",
  reason: "customer_request",
});

// Later...
await commet.subscriptions.resume({
  customerId: "cus_abc123",
});
// Billing resumes, new period starts, pricing is whatever is current
```

Pausing does not freeze the price. When the customer resumes, they get the price in effect at that time.

## Cancellation

Two modes:

**Immediate** -- access ends now, unused portion may be credited.

```typescript
await commet.subscriptions.cancel({
  customerId: "cus_abc123",
  mode: "immediate",
});
```

**End of period** -- access continues until the current billing period ends. This is the more customer-friendly option and the most common.

```typescript
await commet.subscriptions.cancel({
  customerId: "cus_abc123",
  mode: "end_of_period",
});
```

## Reactivation

A canceled customer can subscribe again, but it creates a new subscription. No intro offers (they had a subscription before). Deprecated plans cannot be re-subscribed.

## Key Principles

1. **Grant access on `active`, `trialing`, and `past_due`.** Don't cut off customers during grace periods.
2. **Use webhooks over polling.** State changes happen asynchronously (retries, trial expiry, billing cron). Polling can miss transitions or introduce delays.
3. **Currency is immutable after first payment.** Auto-detected from billing address at checkout. Plan accordingly.
4. **Trials expire at midnight UTC.** The billing engine has a separate query for expired trials independent of the normal billing day check.

## Related

- [Failed Payments](./failed-payments.md) - What happens when payment fails during `active` or after trial
- [Plan Changes](./plan-changes.md) - How upgrades and downgrades affect subscription state
- [Invoicing](./invoicing.md) - When invoices are generated in each state
