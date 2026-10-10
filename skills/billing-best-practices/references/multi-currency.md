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
Does the selected price have a Market override for Brazil?
  |
  +--> Yes --> Charge the Market's price and currency
  |
  +--> No --> Does the plan have regional pricing for Brazil (BRL)?
                |
                +--> Yes --> Charge in BRL at the regional price
                |
                +--> No --> Charge in USD at the base price
```

In Commet Sandbox, the public **Country** control overrides request-country resolution so you can test another country or Market. Live checkout ignores that override. Billing address remains relevant to tax and payment processing, but it does not select the price.

Regional pricing is configured per currency and applies to every country that uses that currency. Each value is either derived from the USD value at the exchange rate you set or stored as a manual override. Markets are optional and available to Enterprise organizations: they group countries that need an explicit price and currency different from their regional default. A non-USD Market price requires regional pricing for that currency on the plan first.

```typescript
import { Commet } from "@commet/node";

const commet = new Commet({ apiKey: process.env.COMMET_API_KEY! });

// Create a plan with a USD base price
const plan = await commet.plans.create({
  name: "Pro",
  code: "pro",
});

const price = await commet.plans.addPrice({
  id: plan.id,
  billingInterval: "monthly",
  price: 9900, // $99.00 USD (base)
  isDefault: true,
});

// Price every BRL country: a manual R$299.00 override, other values derived at the rate
await commet.plans.setRegionalPricing({
  id: plan.id,
  currency: "brl",
  exchangeRate: 5.4,
  prices: [{ priceId: price.id, price: 29900 }],
});

// Optional: a reusable Market overrides price and currency for a country group
const southernCone = await commet.markets.create({
  name: "Southern Cone",
  countryCodes: ["BO", "PY", "UY"],
});

await commet.plans.updatePrice({
  id: plan.id,
  priceId: price.id,
  marketPrices: [
    { marketGroupId: southernCone.id, currency: "usd", price: 3900 },
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

| Currency | Code | Zero-Decimal | Countries |
|----------|------|-------------|-----------|
| US Dollar | USD | No | United States, default |
| Euro | EUR | No | Eurozone countries |
| British Pound | GBP | No | United Kingdom |
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
| Japanese Yen | JPY | Yes | Japan |
| Chinese Yuan | CNY | No | China |
| South Korean Won | KRW | Yes | South Korea |
| Hong Kong Dollar | HKD | No | Hong Kong |
| Singapore Dollar | SGD | No | Singapore |
| New Taiwan Dollar | TWD | No | Taiwan |
| Indian Rupee | INR | No | India |
| Thai Baht | THB | No | Thailand |

## Zero-Decimal Currencies

Most currencies use two decimal places ($99.99 = 9999 cents). Some don't use decimals at all.

| Currency | How $100 is stored | Notes |
|----------|-------------------|-------|
| USD | `10000` | 100.00 in cents |
| CLP | `100` | No decimal places |
| PYG | `100` | No decimal places |
| JPY | `100` | No decimal places |
| KRW | `100` | No decimal places |

This matters for every calculation: proration, overage pricing, invoice totals. Your billing system must know which currencies are zero-decimal and handle them correctly. Failing to do this means charging 100x the intended amount (or 1/100th).

```typescript
// Zero-decimal amounts are passed through as-is.
// 19900 CLP means 19,900 CLP, not 199.00.
await commet.plans.setRegionalPricing({
  id: plan.id,
  currency: "clp",
  exchangeRate: 950,
  prices: [{ priceId: price.id, price: 19900 }],
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

Don't let exchange-rate conversion decide your customer-facing prices. Commet can derive regional values from a rate you set, but override the prices customers see with deliberate local amounts that reflect local purchasing power.

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

If the request country has no Market override on the selected price, the plan's regional price for that country applies. If the plan has no regional pricing for the country's currency, the system falls back to the base USD amount.

This means you can roll out regional pricing incrementally -- start with your largest non-USD currencies and add Markets only where a country group needs a different price.

## Related

- [Tax Compliance](./tax-compliance.md) - Tax calculation works alongside currency detection
- [Proration](./proration.md) - All proration calculated in the subscription's currency
- [Invoicing](./invoicing.md) - Invoice amounts in subscription currency
