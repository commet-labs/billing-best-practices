# Failed Payments

Failed payments are the largest source of involuntary churn in SaaS. Most payment failures are recoverable -- expired cards, insufficient funds, bank holds -- but only if you handle them well. A good dunning flow recovers 50-70% of failed payments automatically.

## What Happens When a Payment Fails

```
Payment fails
  |
  v
Subscription --> past_due (access maintained)
  |
  v
Automatic retries (with backoff)
  |
  +--> Payment succeeds --> active (back to normal)
  |
  +--> All retries exhausted --> canceled (access revoked)
```

The customer keeps full access during the grace period. This is intentional -- cutting off access immediately punishes customers for problems that are usually temporary and outside their control.

## Retry Schedule

Space retries to account for common recovery patterns: bank holds clear within hours, card limit resets happen overnight, and customers update payment methods within a few days.

| Attempt | Timing | Why |
|---------|--------|-----|
| 1st retry | Same day (hours later) | Bank holds, temporary declines |
| 2nd retry | Same day (end of day) | Daily limit resets |
| Final | If not recovered | Subscription canceled |

```typescript
import { Commet } from "@commet/node";

const commet = new Commet({ apiKey: "sk_live_..." });

// Check subscription status after a payment failure event
app.post("/webhooks/commet", async (req, res) => {
  const event = commet.webhooks.verifyAndParse({
    rawBody: req.body,
    signature: req.headers["x-commet-signature"],
    secret: process.env.COMMET_WEBHOOK_SECRET,
  });

  switch (event.type) {
    case "payment.failed":
      await notifyCustomerPaymentFailed(event.data.customerId);
      break;

    case "payment.recovered":
      await clearPaymentWarning(event.data.customerId);
      break;

    case "subscription.canceled":
      if (event.data.reason === "payment_failed") {
        await handleInvoluntaryChurn(event.data.customerId);
      }
      break;
  }
});
```

## Customer Communication

The emails you send during a payment failure directly impact recovery rates. Be helpful, not threatening.

### Email Sequence

**Email 1: Payment failed (immediately)**
- Subject: "Your payment didn't go through"
- Tone: helpful, no urgency
- Content: what happened, link to update payment method
- Don't mention cancellation yet

**Email 2: Reminder (before final retry)**
- Subject: "Action needed: update your payment method"
- Tone: slightly more urgent
- Content: remind them of what they'll lose, link to update payment method

**Email 3: Final notice (after cancellation)**
- Subject: "Your subscription has been canceled"
- Tone: factual, helpful
- Content: what happened, how to resubscribe, what they lose

### Communication Principles

1. **Link directly to payment update.** Every email should have a one-click path to fix the issue.
2. **Show what they'll lose.** "Your team's 3 active projects will become read-only" is more effective than "your subscription will be canceled."
3. **Don't blame the customer.** "Your payment didn't go through" not "your card was declined."

## Grace Period Design

The grace period is the time between the first failure and cancellation. During this time the customer has full access while retries happen and they have the opportunity to update their payment method.

**Best practices:**
- Keep access during the entire grace period
- Show an in-app banner ("Payment issue -- update your card") but don't block usage
- Make the payment update flow as short as possible (one screen, pre-filled card form)

```typescript
// Show a banner for past_due customers
async function getSubscriptionBanner(customerId: string) {
  const { data: subscription } = await commet.subscriptions.getActive({ customerId });

  if (subscription?.status === "past_due") {
    return {
      type: "warning",
      message: "We couldn't process your payment. Please update your payment method.",
      action: subscription.updatePaymentUrl,
    };
  }

  return null;
}
```

## What Happens to Usage During Failure

| Consumption Model | During Grace Period | After Cancellation |
|-------------------|--------------------|--------------------|
| Metered | Usage continues to accumulate | Usage stops |
| Credits | Plan + purchased credits maintained | Plan credits = 0, purchased credits preserved |
| Balance | Balance maintained | Balance = 0 |

Purchased credits survive cancellation. When a customer reactivates, their purchased credits are restored.

## Involuntary Churn Prevention

Beyond retry logic and emails, there are structural decisions that reduce involuntary churn:

**Before the failure:**
- Send "card expiring soon" emails 30 days before expiration
- Support card updater services (Stripe does this automatically)
- Accept multiple payment methods

**During the failure:**
- Retry with smart timing (not just fixed intervals)
- Keep access during grace period
- Make payment update frictionless

**After cancellation:**
- Send a "we miss you" email with a direct resubscribe link
- Preserve purchased credits so reactivation feels seamless
- Don't delete their data immediately -- give them a window to come back

## Pre-Dunning: Card Expiry Warnings

The cheapest payment failure to recover is the one that never happens.

```typescript
// Listen for upcoming card expirations
app.post("/webhooks/commet", async (req, res) => {
  const event = commet.webhooks.verifyAndParse({
    rawBody: req.body,
    signature: req.headers["x-commet-signature"],
    secret: process.env.COMMET_WEBHOOK_SECRET,
  });

  if (event.type === "payment_method.expiring") {
    await sendCardExpiryWarning(
      event.data.customerId,
      event.data.expiresAt
    );
  }
});
```

## Related

- [Subscription Lifecycle](./subscription-lifecycle.md) - How `past_due` fits into the state machine
- [Billing Checklist](./billing-checklist.md) - Pre-launch checks for payment failure handling
- [Invoicing](./invoicing.md) - How failed payment invoices are tracked
