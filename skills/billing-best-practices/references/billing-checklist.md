# Billing Checklist

A pre-launch checklist for SaaS billing. Go through each section before accepting real payments. Billing bugs are the most expensive kind -- they either cost you money or damage customer trust.

## Sandbox Testing

- [ ] **Create a test subscription end-to-end.** Customer creation, plan selection, checkout, payment, active subscription. Verify every step.
- [ ] **Test each plan.** Free, paid monthly, paid yearly, trial. Each path has different logic.
- [ ] **Test with real card numbers.** Use Stripe test cards (4242... for success, 4000... for decline). Don't skip this.
- [ ] **Verify invoice generation.** After a billing cycle, check that invoices contain the correct line items, amounts, and tax.
- [ ] **Test the full billing cycle.** Advance the clock in test mode and verify that recurring invoices are generated correctly.

```typescript
import { Commet } from "@commet/node";

const commet = new Commet({ apiKey: "sk_test_..." });

// Create a test customer
const customer = await commet.customers.create({
  name: "Test Customer",
  email: "test@example.com",
  id: "test_user_001",
});

// Create a subscription
const subscription = await commet.subscriptions.create({
  customerId: customer.id,
  planId: "plan_pro_monthly",
});

// Verify the subscription is in the expected state
console.log(subscription.status); // "pending_payment" or "active" (free)
```

## Webhook Endpoints

- [ ] **Webhook URL is configured and reachable.** Test with a ping event.
- [ ] **Verify signature validation.** Reject requests with invalid signatures.
- [ ] **Handle all critical events:**
  - `subscription.activated`
  - `subscription.canceled`
  - `subscription.past_due`
  - `payment.failed`
  - `payment.recovered`
  - `invoice.created`
- [ ] **Idempotent handlers.** Processing the same event twice should not create duplicate side effects.
- [ ] **Return 200 quickly.** Do heavy processing async. Webhook timeouts cause retries.
- [ ] **Log every event.** You'll need this for debugging billing issues in production.

```typescript
app.post("/webhooks/commet", async (req, res) => {
  const event = commet.webhooks.verifyAndParse({
    rawBody: req.body,
    signature: req.headers["x-commet-signature"],
    secret: process.env.COMMET_WEBHOOK_SECRET,
  });

  // Return 200 immediately, process async
  res.status(200).send("ok");

  // Process the event
  await processEvent(event);
});
```

## Failed Payment Handling

- [ ] **Grace period is configured.** Customers should keep access during retries.
- [ ] **Retry schedule makes sense.** Not too aggressive (annoys payment processors), not too slow (loses recovery window).
- [ ] **Customer notification emails work.** Test the full sequence: first failure, reminder, cancellation notice.
- [ ] **Payment update flow works.** Click the link in the failure email and verify the customer can update their card in one step.
- [ ] **In-app banner shows for past_due customers.** They should see a clear, non-blocking warning with a link to fix payment.
- [ ] **Cancellation after exhausted retries works.** Access is revoked, data is preserved, customer can resubscribe.

## Plan Changes

- [ ] **Upgrade works and is charged correctly.** Verify the prorated invoice has the right credit and charge amounts.
- [ ] **Downgrade schedules for end of period.** Customer keeps current plan until renewal.
- [ ] **Interval change (monthly to yearly) works.** Treated as upgrade, applied immediately.
- [ ] **Seats are prorated on upgrade.** Extra seats on the old plan generate credit.
- [ ] **Usage overage is billed on upgrade.** Pending overage is charged at the old plan's rate.
- [ ] **Warning messages show for downgrades.** If seats or usage exceed the new plan's limits, the customer sees a clear warning before confirming.

## Tax Setup

- [ ] **Tax registrations are configured.** You must register in each jurisdiction where you have nexus before collecting tax.
- [ ] **Product tax code is correct.** SaaS typically uses `txcd_10103001` in Stripe.
- [ ] **Tax appears on test invoices.** Create a test subscription with a taxable address and verify tax is calculated.
- [ ] **Tax-exempt customers handled.** If you have B2B customers, verify VAT ID validation and reverse charge invoicing.
- [ ] **Refund reverses tax.** When you refund a payment, verify that the tax amount is also reversed.

## Customer Portal

- [ ] **Customers can view their invoices.** List and download.
- [ ] **Customers can update their payment method.** Without contacting support.
- [ ] **Customers can upgrade/downgrade.** With clear pricing and proration preview.
- [ ] **Customers can cancel.** With clear messaging about what they'll lose and when.
- [ ] **Customers can view their usage.** Current period usage, included amounts, overage.

## Multi-Currency (If Applicable)

- [ ] **Regional prices are set.** Round numbers in each currency, not just exchange rate conversions.
- [ ] **Currency detection works.** Test with billing addresses from different countries.
- [ ] **Zero-decimal currencies are handled.** CLP and PYG don't use decimal places.
- [ ] **Invoices show the correct currency.** All amounts in the subscription's currency.
- [ ] **Proration works in non-USD currencies.** Verify the math with a non-USD test subscription.

## Usage Tracking (If Applicable)

- [ ] **Usage events are being recorded.** Send test events and verify they appear in the billing system.
- [ ] **Included amounts are correct.** Verify that the first N units are free, then overage kicks in.
- [ ] **Overage pricing is correct.** Send events beyond the included amount and check the invoice.
- [ ] **Usage resets at period boundaries.** Verify that monthly resets happen correctly, especially for quarterly/yearly plans.
- [ ] **Idempotent event tracking.** Sending the same event twice should not double-count.

```typescript
// Test usage tracking
await commet.usage.track({
  customerId: "cus_test",
  featureId: "api_calls",
  quantity: 1,
  idempotencyKey: "req_abc123",
});
```

## Credits and Balance (If Applicable)

- [ ] **Credits block when exhausted.** Access is denied immediately when plan credits + purchased credits = 0.
- [ ] **Credit pack purchase works.** Customer buys a pack, credits are available immediately.
- [ ] **Purchased credits survive cancellation.** Cancel and reactivate, verify purchased credits are restored.
- [ ] **Balance top-up works.** Customer adds funds, balance is updated immediately.
- [ ] **Balance blocking/overage works.** Test both `blockOnExhaustion: true` (blocks immediately) and `false` (allows negative balance, charges at period end).

## Monitoring and Alerts

- [ ] **Billing cron is running.** Verify it executes on schedule.
- [ ] **Failed payment alerts are configured.** You should know when a payment fails, not just the customer.
- [ ] **Invoice generation errors are monitored.** A billing cycle that fails silently is worse than one that fails loudly.
- [ ] **Revenue dashboard shows correct numbers.** Verify totals match what Stripe shows.

## Before You Go Live

1. Run through every path in sandbox mode with test data.
2. Verify invoices match expected amounts to the penny.
3. Test payment failures, retries, and cancellation end-to-end.
4. Confirm webhooks are processing reliably.
5. Check tax calculations for your key markets.
6. Verify the customer portal works for all self-service actions.
7. Switch API keys from test to live.
8. Monitor the first real billing cycle closely.

## Related

- [Subscription Lifecycle](./subscription-lifecycle.md) - States and transitions to test
- [Failed Payments](./failed-payments.md) - Dunning flow to verify
- [Tax Compliance](./tax-compliance.md) - Tax registration requirements
- [Multi-Currency](./multi-currency.md) - Currency-specific testing
