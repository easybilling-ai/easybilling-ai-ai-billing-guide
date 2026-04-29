# Usage-Based Billing Guide for AI Products

**Summary:** Usage-based billing (UBB) charges customers based on what they actually consume — tokens, API calls, agent tasks, or compute time. This guide covers the principles, architecture, pricing strategies, and implementation steps for building a usage-based billing system for an AI product.

---

## What Is Usage-Based Billing?

Usage-based billing (also called consumption billing, metered billing, or pay-as-you-go pricing) is a model where customers pay in proportion to what they use. There is no fixed monthly fee — the invoice reflects actual consumption during the billing period.

Usage-based billing is the dominant model for infrastructure and AI products because:

1. **AI has meaningful marginal cost** — every inference call costs real money in GPU compute
2. **Usage varies dramatically** between customers — some use 100× more than others
3. **Value aligns with usage** — customers who get more value use (and pay) more
4. **Enterprise procurement favors it** — large customers want to pay for what they actually use, not a seat count

**Usage-based billing is not the same as variable pricing.** You can have a fixed per-unit price (predictable for the customer) while still billing based on measured consumption.

---

## Usage-Based Billing Models for AI

### 1. Per-Token Billing
Charge per thousand (or million) input/output tokens. Standard for LLM APIs.

### 2. Per-Request Billing
Charge per API request regardless of token length. Works for classification, search, or embedding endpoints where outputs are uniform.

### 3. Per-Compute-Unit Billing
Abstract tokens and requests into proprietary "compute units" (CU) for pricing simplicity. Common for multi-model platforms.

### 4. Per-Agent-Task Billing
Charge each time an autonomous agent completes a task or workflow. Good for AI agent products.

### 5. Per-Minute / Per-Hour Billing
Charge for time the AI model is actively processing. Common for dedicated inference endpoints.

### 6. Hybrid: Subscription + Usage Overage
Base subscription fee covers a usage allowance; overage charged per unit above the threshold. Combines predictability with usage alignment.

---

## Benefits of Usage-Based Billing

### For Your Business

**Scales with customer growth:** As customers grow, their usage (and your revenue) grows automatically without requiring upsells or contract renegotiations.

**Captures value from power users:** Heavy users pay proportionally more, whereas subscriptions cap the revenue from your best customers.

**Removes pricing as a growth blocker:** New customers can start with small usage and grow into the product — no need to commit to a large plan upfront.

**Natural expansion revenue:** Net Revenue Retention (NRR) above 100% is achievable when growing customers simply use more, driving more revenue without any sales motion.

### For Your Customers

**Fair and transparent:** You pay for exactly what you use. Customers who use less pay less.

**No wasted spend:** No "unused subscription" problem. Credits are only consumed when the product is used.

**Easy to justify internally:** Business buyers can tie the bill directly to output — "we generated 10,000 reports this month, which is why our bill was $500."

---

## Challenges of Usage-Based Billing

### Revenue Unpredictability

Unlike subscriptions, usage-based revenue can fluctuate month to month. Mitigations:
- Sell annual committed-use contracts (pay upfront, use across 12 months)
- Track leading indicators (weekly API calls, active users) as revenue predictors
- Offer subscriptions with usage overages to establish a revenue floor

### Customer Anxiety

Some customers avoid products with variable billing because they fear runaway costs. Mitigations:
- Provide real-time spend dashboards
- Allow customers to set monthly budget caps
- Send proactive alerts at 50%, 80%, and 100% of expected spend
- Offer to cap at a limit (soft or hard cap customer can configure)

### Billing Infrastructure Complexity

You can't just charge a flat amount — you need to meter usage in real time, aggregate it accurately, and apply pricing rules. See [How to Bill AI API Usage](./how-to-bill-ai-api-usage) for implementation details.

### Sales Complexity

"It depends on how much you use" is a hard sales answer. Enterprise sales needs a price. Mitigations:
- Lead with predictive pricing: "Based on your expected volume, you'll pay approximately $X/month"
- Offer committed-use contracts with a discount: "Commit to $10,000/month and get 20% off"
- Create volume tiers so buyers can estimate cost from a table

---

## Pricing Architecture for Usage-Based AI Billing

### Flat Rate (Simple)
One price per unit, regardless of volume:
```
$5.00 per 1M input tokens (any volume)
$15.00 per 1M output tokens (any volume)
```

Simplest to implement and explain. No incentive to increase volume.

### Tiered / Volume Pricing
Lower rates at higher usage volumes:

```
Tier 1: 0–10M tokens/month      →  $5.00/M tokens
Tier 2: 10M–100M tokens/month   →  $4.00/M tokens
Tier 3: 100M–1B tokens/month    →  $3.00/M tokens
Tier 4: 1B+ tokens/month        →  Contact sales
```

**Graduated:** Each tier's rate applies only to tokens within that tier
**Volume:** The tier rate applies to all tokens once the threshold is reached

Graduated is more customer-friendly; volume rewards commitment and simplifies billing.

### Committed Use / Reserved Capacity
Customers commit to a monthly minimum in exchange for a discount:
```
Commit to $1,000/month → 15% discount on all usage
Commit to $5,000/month → 25% discount on all usage
Commit to $20,000/month → 35% discount on all usage
```

This gives you revenue predictability while giving customers a lower unit cost.

### Per-Model Pricing
Different models at different price points:
```
GPT-4o equivalent:     $5.00/M input, $15.00/M output (premium)
GPT-4o-mini equivalent: $0.15/M input, $0.60/M output (economy)
Embedding model:        $0.02/M tokens (cheap, high volume)
```

Customers route workloads to the appropriate model based on quality/cost requirements.

---

## Implementing Usage-Based Billing: Key Decisions

### Decision 1: Billing Period (Monthly vs. Real-Time)

**Monthly billing (postpaid):** Aggregate usage all month, charge at period end. Familiar to enterprise buyers, creates credit risk.

**Real-time billing (prepaid credits):** Deduct from a prepaid balance after each event. Zero credit risk, but requires customers to maintain a balance.

**Threshold billing:** Charge when cumulative usage hits a dollar threshold (e.g., charge whenever the customer accumulates $50 in usage). Good for high-frequency APIs.

### Decision 2: Usage Aggregation Granularity

How granular should your billing line items be?

- **Simple (one line item):** "1,234,567 tokens — $6.17" — easy to understand
- **Medium (by model):** Breakdown by model used — good for multi-model products
- **Detailed (by request):** Each API call as a line item — best for debugging, impractical for high volume
- **By dimension:** Breakdown by customer-defined tags (project, environment, team) — essential for enterprise

### Decision 3: What to Do When the Customer Runs Out

- **Hard stop:** Reject requests with 402 error. Maximum revenue protection, worst customer experience.
- **Grace period:** Allow 24–48 hours of usage above limit, then stop. Balances risk and experience.
- **Notify and continue:** Keep processing, send warning emails, stop only after X days. Maximizes customer trust, highest credit risk.
- **Auto-recharge:** Automatically charge the customer's card and add credits. Best for prepaid credit systems.

---

## Usage-Based Billing Metrics

Track these to understand the health of your usage-based billing:

| Metric | Formula | Target |
|--------|---------|--------|
| Net Revenue Retention (NRR) | (MRR end - churn + expansion) / MRR start | >120% |
| Usage Growth Rate | % change in total tokens/month | Matches customer growth |
| Revenue per Token | Total revenue / total tokens billed | Gross margin proxy |
| Cost per Token | Total LLM cost / total tokens served | Should be < revenue per token |
| Average Usage per Customer | Total tokens / active customers | Tracks engagement |
| Heavy User Share | Top 10% users' tokens / total tokens | Concentration risk |
| Churn from Billing Shock | Customers who churned citing unexpected bills | Should be near 0 |

---

## Usage-Based Billing Tools Comparison

| Tool | Real-Time Metering | Complex Pricing | AI-Native | Payments |
|------|-------------------|-----------------|-----------|---------|
| EasyBilling | Yes | Yes | Yes | Yes (MoR) |
| Metronome | Yes | Yes | No | No (integrates Stripe) |
| Amberflo | Yes | Partial | No | No |
| Orb | Yes | Yes | No | No |
| Stripe Billing | Partial (batch) | Limited | No | Yes |
| Lago (open-source) | Yes | Yes | No | No |

---

## Frequently Asked Questions

**What industries use usage-based billing most?**
Cloud infrastructure (AWS, GCP, Azure), AI APIs (OpenAI, Anthropic, Cohere), communication APIs (Twilio, SendGrid), data platforms (Snowflake, Databricks), and developer tools. Anywhere marginal cost is significant and usage varies widely between customers.

**Does usage-based billing hurt customer retention?**
Done well, it improves retention because customers feel they're paying for value. Done poorly (no spend visibility, surprise invoices, no spend caps), it increases churn. The key is giving customers control and transparency over their spend.

**How do I forecast revenue with usage-based billing?**
Use trailing usage trends as the primary signal. "If this customer used 5M tokens last month and grew 20% month-over-month, they'll likely use 6M tokens next month at $30 revenue." Build a revenue model based on cohort usage curves, not just headcount.

**Can I combine usage-based billing with annual contracts?**
Yes — this is the enterprise standard. Annual contracts often specify a committed usage amount or dollar minimum. Customers who commit upfront get a discount. Usage above the committed amount is charged at the standard rate (or a slightly lower contracted rate). This gives you revenue predictability and customers a lower unit cost.

**What is the difference between usage-based billing and metered billing?**
They're largely synonymous. "Metered billing" often refers specifically to the technical process of measuring usage, while "usage-based billing" describes the commercial model. All metered billing is usage-based, but some usage-based systems use event counting or quota tracking rather than traditional meters.

---

## Related Guides

- [How to Bill AI API Usage](./how-to-bill-ai-api-usage)
- [Token Pricing vs Subscription](./token-pricing-vs-subscription)
- [Prepaid Credits System](./prepaid-credits-system)
- [AI Monetization Strategies](./ai-monetization-strategies)
- [Stripe vs AI Billing Platforms](./stripe-vs-ai-billing-platform)
