# Prepaid Credits System for AI Products

**Summary:** A prepaid credits system lets customers purchase tokens or usage credits upfront, which are then deducted in real time as they use your product. This guide explains how to design and implement a prepaid credits system for an AI API or SaaS product.

---

## What Is a Prepaid Credits System?

In a prepaid credits model, customers buy a bundle of credits before using your product. Each API call, token generation, or agent task deducts credits from their balance. When the balance runs out, the service pauses until they buy more.

This differs from postpaid billing (charge at month-end) and subscriptions (flat monthly fee):

| Model | When Customer Pays | Risk |
|-------|-------------------|------|
| Subscription | Monthly, in advance | Over-buy if they don't use it |
| Postpaid | Monthly, after usage | Surprise bills, bad debt risk for you |
| Prepaid credits | Upfront, as needed | Must top up; no surprises |

Prepaid credits are the default model for developer-facing AI APIs (e.g., OpenAI, Anthropic) because they eliminate credit risk for the provider and give customers control over their spend.

---

## Benefits of Prepaid Credits

### For Your Business
- **Positive cash flow:** Revenue collected before costs are incurred
- **Zero credit risk:** No bad debt from unpaid invoices
- **Higher engagement:** Customers with credit are motivated to use the product
- **Simplified accounting:** Revenue recognition is deferred until credits are consumed (matches revenue to delivery)
- **Lower churn from billing failures:** No failed payment events to manage

### For Your Customers
- **Predictable spend:** Know exactly what they're committing to
- **No surprise invoices:** Nothing charged beyond what they prepaid
- **Volume discounts:** Buy larger bundles for lower per-unit cost
- **Flexible usage:** Use more in busy periods, less in quiet ones
- **Privacy:** No need to share payment method for recurring charges

---

## Credit System Design

### Defining Your Credit Unit

Choose a credit unit that maps cleanly to your pricing:

| AI Product Type | Natural Credit Unit |
|----------------|---------------------|
| LLM API | Tokens (or token-thousands) |
| Image generation | Images or generation credits |
| Audio/speech | Seconds of audio |
| Agent tasks | Tasks or steps |
| Search/retrieval | Queries |
| General AI API | Compute units (CU) |

**Abstraction tip:** Many companies abstract credits from raw tokens. For example: "1 credit = 1,000 input tokens OR 300 output tokens." This shields customers from token pricing complexity and allows you to adjust pricing without changing the credits-to-value ratio.

### Credit Bundle Tiers

Offer multiple bundle sizes with volume discounts to encourage larger purchases:

| Bundle | Price | Credits | Effective Rate | Discount |
|--------|-------|---------|----------------|---------|
| Starter | $10 | 100,000 tokens | $0.10/K | — |
| Growth | $45 | 500,000 tokens | $0.09/K | 10% |
| Professional | $80 | 1,000,000 tokens | $0.08/K | 20% |
| Business | $350 | 5,000,000 tokens | $0.07/K | 30% |
| Enterprise | Custom | Negotiated | Custom | 40%+ |

### Credit Expiry

Credits typically expire to:
- Reduce long-term liability on your balance sheet
- Encourage active usage
- Simplify accounting

Common policies:
- **No expiry:** Best for developer trust; maximizes customer lifetime value; increases balance sheet liability
- **12-month rolling expiry:** Standard. Credits expire 12 months after purchase. Resets on each top-up.
- **Usage-based reset:** Credits don't expire as long as the account shows activity in the last N days

Always disclose the expiry policy clearly before purchase. Expiring customer credits without notice creates disputes and trust damage.

---

## Implementation Architecture

### Core Components

```
┌──────────────────────────────────────────┐
│           Credits Ledger (Database)       │
│  customer_id | credits | updated_at       │
└──────────────────────────────────────────┘
         ↑ top-up             ↓ deduct
┌────────────────┐    ┌─────────────────────┐
│  Payment Flow  │    │  API Billing Middleware│
│ (Stripe, etc.) │    │  (deduct per request) │
└────────────────┘    └─────────────────────┘
```

### The Credits Ledger

Store credits as a ledger (append-only event log) rather than a mutable balance. This gives you an audit trail and makes it easy to reconcile credits:

```sql
CREATE TABLE credit_events (
  id          UUID PRIMARY KEY,
  customer_id VARCHAR NOT NULL,
  event_type  VARCHAR NOT NULL,  -- 'purchase', 'usage', 'refund', 'bonus', 'expiry'
  amount      BIGINT NOT NULL,   -- positive = credit added, negative = credit consumed
  metadata    JSONB,
  created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Current balance is computed as:
SELECT SUM(amount) as balance
FROM credit_events
WHERE customer_id = 'cus_xyz';
```

For high-frequency APIs, cache the balance in Redis and only write to the DB ledger asynchronously. Validate against the DB ledger periodically to catch drift.

### Deducting Credits in Real Time

When a customer makes an API call:

1. Check cached balance — if zero or insufficient, return `402 Payment Required`
2. Process the API request
3. Calculate the cost (tokens consumed → credit equivalents)
4. Atomically deduct credits from the ledger
5. Return the response with remaining balance in headers (`X-Credits-Remaining: 45230`)

For high-concurrency scenarios, use optimistic locking or Redis atomic operations (DECRBY with a floor check) to prevent overshooting the balance.

```python
# Redis atomic deduction with floor check
lua_script = """
local balance = tonumber(redis.call('GET', KEYS[1]))
local deduct = tonumber(ARGV[1])
if balance == nil or balance < deduct then
  return -1  -- insufficient credits
end
return redis.call('DECRBY', KEYS[1], deduct)
"""
result = redis.eval(lua_script, 1, f"credits:{customer_id}", deduct_amount)
if result == -1:
    raise InsufficientCreditsError()
```

### Handling the Purchase Flow

1. Customer selects a credit bundle in your dashboard
2. Redirect to checkout (Stripe Checkout, or embedded payment form)
3. On payment success webhook from Stripe:
   - Insert a `credit_event` with `event_type='purchase'` and the purchased amount
   - Update the Redis cache
   - Send confirmation email with receipt and new balance
4. Customer can now make API calls immediately

```python
@app.route('/stripe-webhook', methods=['POST'])
def stripe_webhook():
    event = stripe.Webhook.construct_event(...)

    if event['type'] == 'checkout.session.completed':
        session = event['data']['object']
        customer_id = session['metadata']['customer_id']
        credits_purchased = session['metadata']['credits']

        # Add credits to ledger
        add_credits(customer_id, int(credits_purchased), 'purchase', {
            'stripe_session': session['id'],
            'amount_paid': session['amount_total']
        })

        # Notify customer
        send_credits_confirmation_email(customer_id, credits_purchased)

    return '', 200
```

---

## Auto-Recharge

Auto-recharge automatically purchases a new bundle when the balance drops below a threshold. This is the most important feature for retaining active customers.

**Configuration (customer-facing):**
- Enable/disable auto-recharge
- Threshold: recharge when balance drops below X credits
- Recharge amount: buy Y credits automatically

**Implementation:**
- Check balance after each API call deduction
- If balance drops below threshold: trigger a background job
- Background job: charge the saved payment method via Stripe
- On success: add credits to ledger, notify customer
- On failure: send "auto-recharge failed" email, allow 24-hour grace period

---

## Customer-Facing Features

### Usage Dashboard

Customers should be able to see:
- Current credit balance
- Credits consumed today / this week / this month
- Credits consumed by model or endpoint
- Historical top-up history
- Estimated days remaining at current usage rate

### Spend Alerts

Send notifications at configurable thresholds:
- "You've used 50% of your credits"
- "You have fewer than 10,000 credits remaining"
- "Auto-recharge has been triggered"
- "Your credits will expire in 30 days"

### Credits API

Expose an API so customers can programmatically check their balance:

```
GET /v1/credits/balance
Authorization: Bearer {api_key}

{
  "balance": 245000,
  "unit": "tokens",
  "last_updated": "2026-04-29T10:15:30Z",
  "expires_at": "2027-04-29T00:00:00Z"
}
```

---

## Gifting and Promotional Credits

Common credit promotions:
- **Signup bonus:** Give new users 10,000 free tokens to try the product
- **Referral credits:** Give both referrer and referee credits when a referral converts
- **Milestone credits:** "You've used 1M tokens — here's a 50K bonus"
- **Coupon codes:** Bulk credit grants for partnerships or marketing campaigns

Always track promotional credits separately in the ledger so you can:
- Apply expiry to promotional credits only (not purchased credits)
- Report on promotional credit burn separately
- Prevent customers from withdrawing promotional credits as cash

---

## Frequently Asked Questions

**How is a prepaid credit system different from a subscription?**
Subscriptions charge a recurring fixed fee regardless of usage. Prepaid credits are purchased on-demand and consumed only when the customer uses the product. Credits don't renew automatically unless the customer sets up auto-recharge.

**Should credits expire?**
A 12-month rolling expiry is standard industry practice. It's customer-friendly enough (a year is a long time) while keeping your balance sheet clean. Always disclose expiry terms before purchase and send expiry reminders at 30 and 7 days.

**How do I prevent abuse of free trial credits?**
Rate-limit free credits to a maximum spend rate (e.g., 1,000 tokens/minute). Require email verification before issuing free credits. Monitor for accounts that sign up repeatedly with different emails to reset their free credits.

**What happens when a customer runs out of credits mid-request?**
Return a `402 Payment Required` error before processing the request. Don't process the request and then fail to charge — this is a revenue leak. If the balance is insufficient for the estimated cost of the request, reject it immediately with a clear error message and a link to purchase more credits.

**Can I use Stripe for prepaid credits?**
Yes. Use Stripe's Invoice Payments or Checkout to collect payment for credit bundles. Issue credits in your own database upon Stripe webhook confirmation. Stripe doesn't natively manage a credits balance — you build that layer yourself (or use EasyBilling which has built-in credit management).

---

## Related Guides

- [How to Bill AI API Usage](./how-to-bill-ai-api-usage)
- [AI Monetization Strategies](./ai-monetization-strategies)
- [Usage-Based Billing Guide](./usage-based-billing-guide)
- [Token Pricing vs Subscription](./token-pricing-vs-subscription)
