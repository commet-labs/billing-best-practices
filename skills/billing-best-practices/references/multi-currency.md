# Multi-Currency

Selling globally means charging customers in their local currency. This isn't just a nice-to-have -- conversion rates increase significantly when customers see prices in their own currency. But multi-currency billing introduces complexity that you need to handle deliberately.

## How Currency Detection Works

Presentment pricing is resolved from the checkout request country. The flow:

```
Checkout request arrives
  |
  v
Country resolved (e.g., Brazil)
  |
  v
Check: does the selected price have an override for Brazil's Market?
  |
  +--> Yes --> Charge in BRL at the configured price
  |
  +--> No --> Charge in USD at the base price
```

In Commet Sandbox, the public **Country** control overrides request-country resolution so you can test another Market. Live checkout ignores that override. Billing address remains relevant to tax and payment processing, but it is not the Market selector.

```typescript
import { Commet } from "@commet/node";

const commet = new Commet({ apiKey: process.env.COMMET_API_KEY! });

// Create reusable country Markets first
const brazil = await commet.markets.create({
  name: "Brazil",
  countryCodes: ["BR"],
});

const argentina = await commet.markets.create({
  name: "Argentina",
  countryCodes: ["AR"],
});

// Create a plan, then add a base price with explicit Market overrides
const plan = await commet.plans.create({
  name: "Pro",
  code: "pro",
});

const price = await commet.plans.addPrice({
  id: plan.id,
  billingInterval: "monthly",
  price: 9900, // $99.00 USD (base)
  marketPrices: [
    { marketGroupId: brazil.id, currency: "brl", price: 29900 },
    { marketGroupId: argentina.id, currency: "ars", price: 4990000 },
  ],
});
```

## Currency Is Immutable

Once a customer makes their first payment, `subscription.currency` is locked. It cannot change for the lifetime of that subscription. This is a deliberate design decision:

- Invoices stay consistent
- No exchange rate surprises mid-cycle
- Revenue tracking is predictable
- Proration calculations are always in one currency

If a customer moves countries and needs a different currency, they must cancel and create a new subscription.

## Supported Currencies

The approach to regional pricing: define your canonical price in USD, then optionally set local prices for specific markets.

| Currency | Code | Zero-Decimal | Common Markets |
|----------|------|-------------|----------------|
| US Dollar | USD | No | United States, default |
| Euro | EUR | No | EU countries |
| Brazilian Real | BRL | No | Brazil |
| Mexican Peso | MXN | No | Mexico |
| Argentine Peso | ARS | No | Argentina |
| Chilean Peso | CLP | Yes | Chile |
| Colombian Peso | COP | No | Colombia |
| Peruvian Sol | PEN | No | Peru |
| Uruguayan Peso | UYU | No | Uruguay |
| Paraguayan Guarani | PYG | Yes | Paraguay |
| Boliviano | BOB | No | Bolivia |
| Canadian Dollar | CAD | No | Canada |

## Zero-Decimal Currencies

Most currencies use two decimal places ($99.99 = 9999 cents). Some don't use decimals at all.

| Currency | How $100 is stored | Notes |
|----------|-------------------|-------|
| USD | `10000` | 100.00 in cents |
| CLP | `100` | No decimal places |
| PYG | `100` | No decimal places |

This matters for every calculation: proration, overage pricing, invoice totals. Your billing system must know which currencies are zero-decimal and handle them correctly. Failing to do this means charging 100x the intended amount (or 1/100th).

```typescript
// Zero-decimal amounts are passed through as-is.
// 19900 CLP means 19,900 CLP, not 199.00.
const chile = await commet.markets.create({
  name: "Chile",
  countryCodes: ["CL"],
});

await commet.plans.addPrice({
  id: plan.id,
  billingInterval: "yearly",
  price: 99000,
  marketPrices: [
    { marketGroupId: chile.id, currency: "clp", price: 19900 },
  ],
});
```

## Rate Scale vs Settlement Scale

Billing systems need two levels of precision:

| Scale | Precision | Used For | Example ($1.00) |
|-------|-----------|----------|-----------------|
| Rate scale | 10000 = $1.00 | Unit prices, per-unit charges, balance | `10000` |
| Settlement scale | 100 = $1.00 | Plan base price, invoice totals, payments | `100` |

Rate scale enables sub-cent pricing. If you charge $0.001 per API call, that's `10` in rate scale -- impossible to represent in settlement scale.

**Key conversions:**
- Rate to amount: `quantity x rate / 10000` (gives settlement cents)
- Rate to cents: `rate / 100`
- Cents to rate: `cents x 100`

## Regional Pricing Strategy

### Set Local Prices, Don't Convert

Don't dynamically convert USD to local currencies using exchange rates. Instead, set deliberate local prices that reflect local purchasing power.

**Why:**
- Exchange rates fluctuate daily -- your prices shouldn't
- Purchasing power differs from exchange rate (a $99 product isn't "worth" the USD equivalent in BRL)
- Clean round numbers convert better ($299 BRL, not $287.43 BRL)

### Pricing Guidelines

1. **Research local SaaS pricing.** What do comparable products charge in that market?
2. **Use round numbers.** R$299, not R$287.43.
3. **Account for payment method costs.** Some markets have higher transaction fees.
4. **Review periodically.** Currency values shift over time. Update regional prices quarterly or when rates move significantly.

## Non-USD Payment Processing

When a customer pays in a non-USD currency, Commet records the presentment amount and reads settlement data from the provider backend when that provider exposes it:

| Amount | What It Represents |
|--------|--------------------|
| `presentmentAmount` | What the customer paid in their currency |
| `grossAmount` | Provider-reported USD settlement amount, nullable when unavailable |

Provider fees, settlement, and net values remain provider-owned. Do not synthesize a missing USD amount from the presentment amount.

## Fallback Behavior

If the request country has no Market override on the selected price, the system falls back to that price's base amount and currency.

This means you can roll out Market pricing incrementally -- start with your largest non-default markets and expand over time.

## Related

- [Tax Compliance](./tax-compliance.md) - Tax calculation works alongside currency detection
- [Proration](./proration.md) - All proration calculated in the subscription's currency
- [Invoicing](./invoicing.md) - Invoice amounts in subscription currency
