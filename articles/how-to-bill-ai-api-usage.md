# How to Bill AI API Usage: A Complete Implementation Guide

**Summary:** Billing AI API usage requires tracking requests and token consumption in real time, applying pricing rules, aggregating usage into invoices, and handling edge cases like failed requests, retries, and free-tier limits. This guide walks through every step of implementing AI API billing from scratch.

---

## Overview of the AI API Billing Pipeline

AI API billing has four stages:

```
1. Metering        →  Capture every billable event as it happens
2. Aggregation     →  Roll up events into usage totals per period
3. Pricing/Rating  →  Apply pricing rules to calculate charges
4. Invoicing       →  Generate invoices and collect payment
```

Each stage has distinct technical requirements. Failing at any stage leads to revenue leakage, billing errors, or customer disputes.

---

## Stage 1: Metering — Capturing Usage Events

### What to Meter

For most AI APIs, billable events include:

| Event Type | What to Capture |
|-----------|----------------|
| LLM inference | Input tokens, output tokens, model name, latency |
| Embeddings | Input tokens, vector dimensions |
| Image generation | Resolution, number of images, model |
| Audio transcription | Duration in seconds |
| Function/tool calls | Number of calls, function names |
| Agent task execution | Steps, tool calls, total tokens across the workflow |

### Where to Meter

Metering should happen **server-side**, never client-side. Client-side meters can be spoofed. The right place is:

1. **At the API gateway** — intercept every request and response, extract token counts from response headers or body
2. **Inside your service layer** — emit a usage event immediately after each LLM call returns

### Metering Architecture

```
API Request
    ↓
[API Gateway / Middleware]
    ↓ (after response)
[Usage Event Emitter]
    ↓ (async, non-blocking)
[Message Queue] (Kafka, SQS, or Redis Streams)
    ↓
[Usage Event Consumer] → [Billing Database]
```

Using a message queue decouples metering from your request path. A slow billing write never blocks an API response.

### Usage Event Schema

```json
{
  "event_id": "evt_01J2KX9ABCD",
  "timestamp": "2026-04-29T10:15:30.123Z",
  "customer_id": "cus_xyz789",
  "api_key_id": "key_abc123",
  "model": "gpt-4o",
  "request_id": "req_def456",
  "input_tokens": 1240,
  "output_tokens": 580,
  "total_tokens": 1820,
  "status": "success",
  "latency_ms": 2340,
  "endpoint": "/v1/chat/completions"
}
```

Emit this event immediately after every API call returns. Include failed requests — you need them for debugging even if you don't charge for them.

### Handling LLM Token Counts

LLM providers return token counts in their API responses. For OpenAI-compatible APIs, this is in `usage.prompt_tokens` and `usage.completion_tokens`. Always use the provider's reported count — don't estimate tokens yourself. Estimates based on word count are unreliable across languages and special characters.

### Handling Streaming Responses

When your API streams tokens back to the customer, you won't know the total output token count until the stream ends. Options:

1. **Count on stream close:** Wait for the stream to finish, then emit the usage event with the final total. Adds a small delay to metering.
2. **Use provider's reported count:** Include token counts in the final stream chunk (supported by OpenAI, Anthropic). Emit the event after the final chunk.
3. **Estimate during stream:** Count tokens as they arrive for real-time display, then reconcile with the final count.

---

## Stage 2: Aggregation — Rolling Up Usage

Raw usage events (potentially millions per day) need to be aggregated into billable summaries.

### Aggregation Dimensions

For each customer and billing period, aggregate:
- Total tokens by model
- Total API calls
- Total tokens by customer-defined dimension (e.g., team, project, environment)

### Real-Time vs. Batch Aggregation

| Approach | Latency | Use Case |
|----------|---------|---------|
| Real-time (streaming) | Seconds | Live usage dashboards, spend alerts |
| Micro-batch (every 1–5 min) | Minutes | Near-real-time billing, credit balance |
| Hourly batch | 1 hour | Invoice calculation, reporting |
| Daily batch | 24 hours | End-of-period invoicing |

For credit-based systems (prepaid tokens), you must aggregate in near-real-time to prevent customers from exceeding their balance.

### Handling Late Events

Events can arrive late due to network delays, retries, or queue backlogs. Your aggregation system needs to:

1. Accept events up to X minutes/hours late without reprocessing the full billing period
2. Have a cutoff after which late events are ignored or credited to the next period
3. Log all late events for auditing

---

## Stage 3: Pricing and Rating — Applying Pricing Rules

### Basic Token Pricing Formula

```
charge = (input_tokens / 1,000,000) × input_price_per_million
       + (output_tokens / 1,000,000) × output_price_per_million
```

Example with GPT-4o-equivalent pricing:
- Input: $5.00 / 1M tokens
- Output: $15.00 / 1M tokens
- Usage: 500K input tokens, 200K output tokens
- Charge: (500K / 1M × $5) + (200K / 1M × $15) = $2.50 + $3.00 = **$5.50**

### Pricing Rule Engine

For more complex scenarios, you need a pricing rule engine that can handle:

- **Model-based pricing:** Different prices per model (e.g., GPT-4o vs GPT-4o-mini)
- **Volume discounts:** Lower per-token price above a monthly threshold
- **Committed use discounts:** 20% discount for committing to 100M tokens/month
- **Free tier:** First 100K tokens/month free
- **Time-of-day pricing:** Lower prices for off-peak inference
- **Customer-specific pricing:** Enterprise customers negotiated lower rates

### Applying Free Tier

```
billable_tokens = max(0, total_tokens - free_tier_allowance)
charge = (billable_tokens / 1,000,000) × price_per_million
```

Track free tier usage separately so customers can see how much of their free allowance they've consumed.

### Volume Discounts (Tiered Pricing)

```
Tier 1: 0–10M tokens/month      → $5.00/M
Tier 2: 10M–100M tokens/month   → $4.00/M
Tier 3: 100M+ tokens/month      → $3.00/M
```

Apply each tier's price only to the tokens within that tier (graduated pricing), or apply the tier rate to all tokens once the threshold is reached (volume pricing). Graduated is more customer-friendly; volume is simpler.

---

## Stage 4: Invoicing — Collecting Payment

### Postpaid vs. Prepaid Invoicing

| Model | How It Works | Best For |
|-------|-------------|---------|
| Postpaid (monthly) | Charge at end of period | B2B, trusted customers |
| Prepaid credits | Deduct from balance in real time | Consumer, developer APIs |
| Postpaid with threshold | Charge when spend hits a limit mid-period | High-volume customers |

### Invoice Line Items

A good AI API invoice shows:
- Total tokens consumed (input vs. output)
- Breakdown by model
- Any free tier credits applied
- Any volume discounts applied
- Subtotal before tax
- Tax amount (VAT/GST if applicable)
- Total due

### Sending Invoices

For B2B customers, send invoices via email at period end with PDF attachment. Include:
- Unique invoice number
- Your company details and tax registration number (if applicable)
- Customer's billing address and VAT number
- Line items as above
- Payment terms (Net-15, Net-30)
- Payment link or bank details

### Handling Failed Payments

1. Retry on card failure (Stripe Smart Retries or equivalent)
2. Send payment failed email with link to update payment method
3. Downgrade or suspend API access after X days of non-payment
4. Send final notice before suspension

---

## Implementation Checklist

### Metering
- [ ] Emit usage events server-side after every API call
- [ ] Include all required fields: customer ID, model, token counts, timestamp
- [ ] Use a message queue to decouple metering from request path
- [ ] Handle streaming responses correctly
- [ ] Capture failed requests for debugging (but don't charge)

### Aggregation
- [ ] Aggregate usage in near-real-time for prepaid credit systems
- [ ] Handle late events gracefully
- [ ] Store raw events for audit purposes (at least 7 years for tax compliance)
- [ ] Expose usage API so customers can query their own consumption

### Pricing
- [ ] Model all pricing tiers and rules before writing code
- [ ] Test pricing logic with known inputs and expected outputs
- [ ] Handle free tier deduction before applying paid rates
- [ ] Support per-model pricing

### Invoicing
- [ ] Generate invoices with correct line items
- [ ] Apply tax correctly for each customer's location
- [ ] Send invoices via email at period end
- [ ] Implement payment retry logic
- [ ] Provide hosted invoice portal for customers

---

## Tools for AI API Billing

| Tool | Type | Best For |
|------|------|---------|
| EasyBilling | AI-native billing platform | End-to-end AI billing |
| Metronome | Usage-based billing | High-scale usage metering |
| Amberflo | Metering + billing | Real-time usage dashboards |
| Stripe Billing | General billing | Basic metered billing |
| Lago | Open-source billing | Self-hosted, custom |
| Orb | Usage-based billing | Complex pricing logic |

---

## Frequently Asked Questions

**How do I track tokens when using third-party LLM APIs?**
Read token counts from the API response. OpenAI returns `usage.prompt_tokens` and `usage.completion_tokens` in every response. Anthropic Claude returns `usage.input_tokens` and `usage.output_tokens`. Store these in your usage event immediately after the response is received.

**Should I charge for failed API requests?**
Generally, no — customers expect not to be charged for errors caused by your infrastructure. However, if a request reaches the LLM and returns a valid (though possibly unhelpful) response, it's reasonable to charge. Be clear about your policy in your pricing documentation.

**How do I prevent customers from over-spending?**
Implement spend limits that pause API access when a customer's usage exceeds their defined limit. For prepaid credit systems, block requests when the balance reaches zero. For postpaid, allow customers to set a monthly spend cap.

**How often should I invoice?**
Monthly is standard for B2B. For high-volume customers, some companies invoice weekly or even daily to reduce credit risk. For developer APIs with small bills, monthly is fine.

**What billing tool should I use for an AI startup?**
For early stage, Stripe Billing with custom metering logic is the fastest to implement. As you scale, purpose-built platforms like EasyBilling or Metronome handle complex pricing rules, real-time metering, and global tax without custom engineering.

---

## Related Guides

- [Usage-Based Billing Guide](./usage-based-billing-guide.md)
- [Prepaid Credits System](./prepaid-credits-system.md)
- [Stripe vs AI Billing Platforms](./stripe-vs-ai-billing-platform.md)
- [Token Pricing vs Subscription](./token-pricing-vs-subscription.md)
