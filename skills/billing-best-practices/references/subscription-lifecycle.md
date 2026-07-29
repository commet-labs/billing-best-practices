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
     └── customer cancels ──► canceled
```

| State | Access | Billing | Transitions To |
|-------|--------|---------|---------------|
| `draft` | No | None | `pending_payment` |
| `pending_payment` | No | Awaiting payment | `trialing`, `active` |
| `trialing` | Yes | No charge (usage accumulates) | `active`, `canceled` |
| `active` | Yes | Normal billing | `past_due`, `canceled` |
| `past_due` | Yes (grace period) | Retrying payment | `active`, `canceled` |
| `canceled` | No | None | New subscription only |

## One Active Subscription Per Customer

A customer can only have one active subscription at a time. Treat a `409` from subscription creation as the authoritative conflict instead of reproducing a second state machine in application code.

## Creation Flows

### Paid Plan (No Trial)

```typescript
import { Commet } from "@commet/node";

const commet = new Commet({ apiKey: process.env.COMMET_API_KEY! });

// Create a subscription -- customer is redirected to checkout
const subscription = await commet.subscriptions.create({
  customerId: "cus_abc123",
  planId: "pln_pro",
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
  planId: "pln_pro", // plan has trialDays: 14
});

// Status: pending_payment
// Customer completes setup checkout (card saved, no charge)
// Status: trialing, trialEndsAt set to 14 days from activation
// Trial expires --> billing engine charges first invoice --> status: active
```

During the trial: full access, no charge, usage accumulates. At the first paid invoice, a retryable provider decline moves the subscription to `past_due`; a missing payment method or customer-action-required outcome leaves it in `pending_payment` until checkout is completed.

### Free Plan

```typescript
const subscription = await commet.subscriptions.create({
  customerId: "cus_abc123",
  planId: "pln_free",
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

### Query state for access

Query current state when making an access decision. Webhooks are useful for background reactions such as email or provisioning, but a copied local webhook state is not the billing source of truth.

Listen for state change events and update your app in real time. This is the most reliable approach -- you never miss a transition.

```typescript
// In your webhook handler
app.post("/webhooks/commet", async (req, res) => {
  const event = commet.webhooks.verifyAndParse({
    rawBody: req.body,
    signature: req.headers["x-commet-signature"],
    secret: process.env.COMMET_WEBHOOK_SECRET,
  });

  if (!event) {
    res.status(400).send("invalid signature");
    return;
  }

  switch (event.event) {
    case "subscription.activated":
      await grantAccess(event.data.customerId);
      break;

    case "subscription.past_due":
      await showPaymentBanner(event.data.customerId);
      break;

    case "subscription.canceled":
      await revokeAccess(event.data.customerId);
      break;

    case "trial.will_end":
      await sendTrialEndingEmail(event.data.customerId);
      break;
  }
});
```

```typescript
async function checkAccess(customerId: string): Promise<boolean> {
  const subscription = await commet.subscriptions.getActive({ customerId });

  if (!subscription) return false;

  const hasAccess = ["trialing", "active", "past_due"].includes(
    subscription.status
  );

  return hasAccess;
}
```

**Why `past_due` grants access:** The customer is in a grace period while payment is retried. Cutting them off immediately increases churn -- most failed payments are recovered automatically.

## Scheduled cancellation and reversal

```typescript
// Schedule cancellation at period end
await commet.subscriptions.cancel({
  id: "sub_xxx",
  reason: "customer_request",
  immediate: false, // cancels at period end
});

// Later, if customer changes their mind before period ends...
await commet.subscriptions.uncancel({
  id: "sub_xxx",
});
// Cancellation reversed, billing continues as normal
```

Commet does not expose a pause/resume lifecycle. Do not present scheduled cancellation as a pause.

## Cancellation

Two modes:

**Immediate** -- access ends now.

```typescript
await commet.subscriptions.cancel({
  id: "sub_xxx",
  immediate: true,
});
```

**End of period** -- access continues until the current billing period ends. This is the more customer-friendly option and the most common.

```typescript
await commet.subscriptions.cancel({
  id: "sub_xxx",
  // immediate defaults to false -> cancels at period end
});
```

## Reactivation

A canceled subscription can be reactivated through `subscriptions.reactivate()`. Commet charges first and restores state only after payment succeeds.

## Key Principles

1. **Grant access on `active`, `trialing`, and `past_due`.** Don't cut off customers during grace periods.
2. **Query current state for access.** Use webhooks for background reactions.
3. **Currency is immutable after first payment.** Auto-detected from billing address at checkout. Plan accordingly.
4. **Trial conversion uses the exact trial-end instant.**

## Related

- [Failed Payments](./failed-payments.md) - What happens when payment fails during `active` or after trial
- [Plan Changes](./plan-changes.md) - How upgrades and downgrades affect subscription state
- [Invoicing](./invoicing.md) - When invoices are generated in each state
