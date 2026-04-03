# Tax Compliance

Collecting tax on SaaS sales is legally required in most jurisdictions. The question isn't whether to handle tax -- it's how. You have two paths: do it yourself or use a Merchant of Record. The right choice depends on where your customers are and how much compliance overhead you're willing to manage.

## The Two Models

### Merchant of Record (MoR)

A Merchant of Record is a third party that legally sells your product on your behalf. They handle tax collection, remittance, compliance, refunds, and chargebacks. You receive a payout minus their fee.

```
Customer --> MoR (legal seller) --> Tax collected + remitted
                |
                v
            Payout to you (minus fees)
```

**Pros:**
- Zero tax compliance work on your end
- Handles global tax registration automatically
- Manages refunds, chargebacks, VAT invoices
- Compliance scales with your customer base without additional work

**Cons:**
- Higher fees (typically 4-8% of revenue)
- Less control over the checkout experience
- Customer sees the MoR as the seller on receipts

**Best for:** Companies selling globally, small teams without tax/legal resources, products with customers in many tax jurisdictions.

### DIY (Stripe Tax, TaxJar, etc.)

You are the seller. You collect tax using a tax calculation service, then remit it yourself (or use a filing service).

```
Customer --> Your checkout --> Tax calculation API --> Tax collected
                                                          |
                                                          v
                                                   You remit to each jurisdiction
```

**Pros:**
- Lower per-transaction cost
- Full control over checkout and customer experience
- Customer sees your company name on receipts

**Cons:**
- You must register for tax in each jurisdiction where you have nexus
- You must file and remit taxes regularly (monthly or quarterly, per jurisdiction)
- You handle refunds, disputes, and VAT invoices
- Compliance work scales linearly with your customer base

**Best for:** Companies selling primarily in one country, teams with tax/legal support, high volume where the fee savings justify the compliance cost.

## Stripe Tax

Stripe Tax is the most common DIY approach for SaaS. It automatically calculates the right tax amount based on the customer's location, your product type, and your tax registrations.

```typescript
import { Commet } from "@commet/node";

const commet = new Commet({ apiKey: "sk_live_..." });

// Tax is calculated automatically at checkout
// based on the customer's billing address
const subscription = await commet.subscriptions.create({
  customerId: "cus_abc123",
  planId: "plan_pro_monthly",
  // Tax calculated from billing address country/state
});
```

### What Stripe Tax Handles

- Calculates correct tax rate based on customer location
- Applies the right product tax code (SaaS = `txcd_10103001`)
- Handles tax-exempt customers
- Generates tax-compliant invoices
- Tracks your tax liability by jurisdiction

### What Stripe Tax Does NOT Handle

- Tax registration (you must register in each jurisdiction yourself)
- Tax filing and remittance (you must file returns and pay collected tax)
- Determining where you have nexus (you must decide where to register)

## Tax Nexus: Where You Need to Collect

You must collect tax in jurisdictions where you have "nexus" -- a legal obligation to collect. Nexus is created by:

| Nexus Type | Trigger | Example |
|------------|---------|---------|
| Physical | Office, employees, warehouse | You have an employee in California |
| Economic | Revenue or transaction threshold | >$100K in sales to Texas customers |

**US:** Each state has different thresholds (typically $100K in revenue or 200 transactions). Check each state's rules.

**EU:** VAT applies to all B2C digital sales regardless of volume. You must register in at least one EU country (OSS simplifies this to one filing).

**Other:** Canada (GST/HST), Australia (GST), UK (VAT), India (GST) -- each has its own rules.

## Invoice Requirements

Tax-compliant invoices must include:

| Field | Required For |
|-------|-------------|
| Seller name and address | Everywhere |
| Buyer name and address | Most jurisdictions |
| Tax registration number | EU VAT, many others |
| Tax rate applied | Everywhere |
| Tax amount | Everywhere |
| Invoice number (sequential) | EU, many others |
| Date of supply | EU VAT |
| Product description | Most jurisdictions |

## VAT Reverse Charge (B2B in EU)

When selling to a business customer in the EU (who provides a valid VAT ID), you don't charge VAT. The customer self-assesses the tax ("reverse charge"). Your invoice must state "Reverse charge" and include the customer's VAT ID.

```typescript
// When creating a customer with a VAT ID
const customer = await commet.customers.create({
  name: "Acme GmbH",
  email: "billing@acme.de",
  address: {
    country: "DE",
    // ...
  },
  taxId: "DE123456789",
});

// Invoices automatically apply reverse charge when VAT ID is valid
```

## Decision Framework

```
Do you sell to customers in multiple countries?
  |
  +--> No (single country) --> DIY with Stripe Tax is manageable
  |
  +--> Yes
        |
        Do you have tax/legal resources?
          |
          +--> Yes --> DIY is viable, but evaluate the ongoing cost
          |
          +--> No --> Use a Merchant of Record
```

## Common Mistakes

1. **Ignoring tax until you're big.** Tax obligations exist from the first sale in most jurisdictions. Retroactive compliance is painful and expensive.

2. **Collecting tax without registering.** If you collect tax in a state where you're not registered, you may owe it but have no legal mechanism to remit it. Register first.

3. **Using the wrong product tax code.** SaaS is taxed differently from physical goods. Using the wrong code means wrong rates.

4. **Not handling tax-exempt customers.** Government entities, nonprofits, and resellers may be tax-exempt. You need a flow to accept and validate exemption certificates.

5. **Forgetting about refunds.** When you refund a payment, you must also reverse the tax. Your system needs to track tax per invoice line for accurate reversals.

## Related

- [Multi-Currency](./multi-currency.md) - Currency detection works alongside tax calculation
- [Invoicing](./invoicing.md) - Tax appears as a line item on invoices
- [Billing Checklist](./billing-checklist.md) - Tax setup is a pre-launch requirement
