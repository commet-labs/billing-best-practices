```
  ╔═══════════════════════════════════════════╗
  ║   ____ ___ _     _     ___ _   _  ____   ║
  ║  | __ )_ _| |   | |   |_ _| \ | |/ ___|  ║
  ║  |  _ \| || |   | |    | ||  \| | |  _   ║
  ║  | |_) | || |___| |___ | || |\  | |_| |  ║
  ║  |____/___|_____|_____|___|_| \_|\____|   ║
  ║                                           ║
  ║            Best Practices                 ║
  ╚═══════════════════════════════════════════╝
```

# Billing Best Practices Skill

A comprehensive agent skill for building production SaaS billing systems. Covers subscription lifecycle, failed payments, proration, multi-currency, tax compliance, plan changes, invoicing, and pre-launch checklists.

## Installation

```bash
npx skills add commet-labs/billing-best-practices
```

## What This Skill Covers

**Subscription Lifecycle**
- States and transitions (active, trialing, past_due, canceled)
- Webhooks vs polling for state changes
- Grace periods and access control

**Payments & Recovery**
- Dunning flows and retry schedules
- Involuntary churn prevention
- Customer communication during failures

**Pricing & Changes**
- Proration calculations for mid-cycle upgrades
- Upgrade vs downgrade timing
- Consumption models (metered, credits, balance)

**Global Billing**
- Multi-currency with regional pricing
- Zero-decimal currency handling
- Tax compliance and Merchant of Record

**Production Readiness**
- Invoice lifecycle and line item types
- Pre-launch billing checklist
- Sandbox testing strategies

## Structure

```
billing-best-practices/
├── SKILL.md                                  # Start here - routes to the right resource
└── references/
    ├── subscription-lifecycle.md             # States, transitions, webhooks
    ├── failed-payments.md                    # Dunning, retries, grace periods
    ├── proration.md                          # Mid-cycle upgrade/downgrade math
    ├── multi-currency.md                     # Regional pricing, currency detection
    ├── tax-compliance.md                     # MoR, automatic tax, Stripe Tax
    ├── plan-changes.md                       # Upgrades, downgrades, interval changes
    ├── invoicing.md                          # Invoice types, billing cycles, line items
    └── billing-checklist.md                  # Pre-launch checklist
```

## Quick Start

Open `SKILL.md` - it has a routing table that directs you to the right resource based on what you need to build.

## License

MIT
