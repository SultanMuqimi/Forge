# Payments Addon
### Forge Addon | Append to your project's CLAUDE.md when accepting payments

---

## CRITICAL: NEVER HANDLE RAW CARD DATA

**Never accept, process, or store raw credit card numbers, CVVs, or sensitive card details yourself.** This violates PCI DSS and exposes you to massive legal and security risks.

Use a payment processor's hosted checkout or client-side tokenization — the card data goes directly from the user's browser to the processor, never touching your server.

---

## PAYMENT PROVIDERS

### Stripe (Default Recommendation)
- Best developer experience
- Global coverage
- Comprehensive features (subscriptions, invoicing, connect for marketplaces)
- Good documentation and SDKs for every stack

### PayPal
- Use when target users prefer PayPal
- More complex API
- Better for one-time payments than subscriptions

### Local Gateways (GCC / Middle East)
- **Tap Payments** — widely supported in GCC
- **HyperPay** — popular in Saudi Arabia
- **Checkout.com** — enterprise-grade regional support
- **Moyasar** — Saudi-focused
- Choose based on user base location

### Never
- Never build your own payment processing
- Never store card data even briefly
- Never accept payment information via email or insecure channels

---

## STRIPE INTEGRATION PATTERN

### Architecture
```
Customer → Stripe Checkout (hosted) → Webhook → Your Backend → Database
                                            ↓
                                        Your Frontend (status display)
```

### Backend Setup
```python
# Python / FastAPI example
import stripe
stripe.api_key = settings.STRIPE_SECRET_KEY

@router.post("/checkout/session")
async def create_checkout_session(data: CheckoutRequest, user: User = Depends(get_current_user)):
    session = stripe.checkout.Session.create(
        payment_method_types=['card'],
        mode='payment',
        line_items=[{
            'price': data.price_id,
            'quantity': 1,
        }],
        success_url=f"{settings.FRONTEND_URL}/success?session_id={{CHECKOUT_SESSION_ID}}",
        cancel_url=f"{settings.FRONTEND_URL}/cancel",
        customer_email=user.email,
        metadata={'user_id': str(user.id)},
    )
    return {'url': session.url}
```

```typescript
// Node.js / TypeScript example
import Stripe from 'stripe'

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: '2024-04-10',
})

export async function createCheckoutSession(userId: string, priceId: string) {
  return stripe.checkout.sessions.create({
    payment_method_types: ['card'],
    mode: 'payment',
    line_items: [{ price: priceId, quantity: 1 }],
    success_url: `${process.env.FRONTEND_URL}/success?session_id={CHECKOUT_SESSION_ID}`,
    cancel_url: `${process.env.FRONTEND_URL}/cancel`,
    metadata: { user_id: userId },
  })
}
```

### Frontend (Just Redirect)
```typescript
const { url } = await api.post('/checkout/session', { priceId })
window.location.href = url
```

No need for Stripe.js unless using Elements for custom UI.

---

## WEBHOOKS — THE REAL SOURCE OF TRUTH

**Never trust success URLs alone.** Users can close the tab, connection can fail, etc. Webhooks are authoritative.

### Webhook Handler
```python
@router.post("/webhooks/stripe")
async def stripe_webhook(request: Request):
    payload = await request.body()
    signature = request.headers.get('stripe-signature')
    
    try:
        event = stripe.Webhook.construct_event(
            payload, signature, settings.STRIPE_WEBHOOK_SECRET
        )
    except Exception:
        raise HTTPException(status_code=400, detail="Invalid signature")
    
    if event['type'] == 'checkout.session.completed':
        await handle_checkout_completed(event['data']['object'])
    elif event['type'] == 'customer.subscription.updated':
        await handle_subscription_updated(event['data']['object'])
    
    return {"received": True}
```

### Webhook Rules
- **Verify signature** — never skip this
- **Idempotency** — webhooks can fire multiple times, design handlers to handle duplicates
- **Respond fast** (< 5 seconds) — do heavy work async
- **Log everything** — webhook failures are hard to debug otherwise
- **Retry logic** — Stripe retries failed webhooks automatically; ensure your handler is idempotent

---

## DATABASE SCHEMA

### Payments Table
```sql
CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    amount_cents INTEGER NOT NULL,  -- store in cents/smallest unit
    currency CHAR(3) NOT NULL,
    status TEXT NOT NULL,  -- pending, succeeded, failed, refunded
    provider TEXT NOT NULL,  -- stripe, paypal, etc.
    provider_payment_id TEXT NOT NULL UNIQUE,
    metadata JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX ON payments(user_id);
CREATE INDEX ON payments(status);
```

### Key Rules
- **Always store amounts in the smallest unit** (cents, halalas, fils) — never floats
- **Store currency explicitly** — never assume USD
- **Never store card details** — only the provider's payment/transaction ID
- **Track status transitions** — who changed what when

---

## SUBSCRIPTIONS

### Stripe Subscriptions
```python
# Create subscription via Checkout
session = stripe.checkout.Session.create(
    payment_method_types=['card'],
    mode='subscription',
    line_items=[{'price': price_id, 'quantity': 1}],
    # ...
)
```

### Key Events to Handle
- `customer.subscription.created` — activate features
- `customer.subscription.updated` — plan changed
- `customer.subscription.deleted` — deactivate features
- `invoice.payment_succeeded` — renewal successful
- `invoice.payment_failed` — start dunning process

### User State
```sql
ALTER TABLE users ADD COLUMN stripe_customer_id TEXT UNIQUE;
ALTER TABLE users ADD COLUMN subscription_status TEXT;
ALTER TABLE users ADD COLUMN subscription_tier TEXT;
ALTER TABLE users ADD COLUMN subscription_current_period_end TIMESTAMPTZ;
```

Check `subscription_status` on every protected request to premium features.

---

## REFUNDS AND DISPUTES

### Refund Flow
```python
refund = stripe.Refund.create(
    payment_intent=payment_intent_id,
    amount=amount_cents,  # partial or full
    reason='requested_by_customer',
)
```

- Always log refund reasons
- Update your internal payment record
- Notify the user via email
- Some regions have mandatory refund windows (EU: 14 days)

### Disputes
- Handle `charge.dispute.created` webhook
- Alert admin immediately
- Respond via Stripe Dashboard or API with evidence
- Track dispute outcomes — high dispute rates trigger processor issues

---

## TESTING

### Test Mode
- Stripe provides test keys — use them in development and testing
- Test cards: `4242 4242 4242 4242` (success), `4000 0000 0000 0002` (declined)
- Never mix live and test keys in the same environment

### Webhook Testing
- Use `stripe listen` CLI to forward webhooks to localhost during dev
- Have a staging environment with real webhook URLs

### Integration Tests
- Mock Stripe API in unit tests
- Use Stripe's test mode in integration tests
- Test idempotency of webhook handlers specifically

---

## SECURITY

### API Keys
- **Secret key** — server only, never expose to frontend
- **Publishable key** — safe for frontend
- **Webhook signing secret** — server only
- Rotate keys if ever compromised

### Fraud Prevention
- Stripe Radar is built-in — enable it
- Consider 3D Secure for high-value transactions
- Rate limit payment attempts per user
- Alert on unusual patterns (many failed attempts, geographic anomalies)

### PCI Compliance
- By using Stripe Checkout or Elements, you qualify for **PCI SAQ A** (simplest)
- Never log card data even if accidentally received
- Keep the payment flow entirely on the provider's domain when possible

---

## UI/UX RULES

### Loading States
- Show clearly that payment is processing
- Never let users click "pay" twice — disable after first click
- Display clear success/failure messages

### Error Messages
- Translate technical errors to human language
- Never expose raw provider error codes to users
- Common errors: insufficient funds, card declined, expired card

### Receipts
- Send email receipts (Stripe does this automatically, or send your own)
- Provide downloadable invoices for businesses
- Include tax breakdowns where required

---

## INTERNATIONAL CONSIDERATIONS

### Multiple Currencies
- Users see prices in their local currency
- Store prices per currency — don't rely on exchange rate conversion
- Display prices with correct decimal places (some currencies have none, like JPY)

### Tax Handling
- Stripe Tax (or alternatives) for automatic tax calculation
- VAT in EU
- Sales tax in US
- GST in GCC (varies by country)
- Consult an accountant for complex multi-country setups

### Local Regulations
- Saudi Arabia: requires local payment gateway for high volume
- EU: PSD2 / SCA compliance
- US: state-specific rules on digital goods

---

## ENVIRONMENT VARIABLES

```
# Stripe
STRIPE_SECRET_KEY=sk_test_...
STRIPE_PUBLISHABLE_KEY=pk_test_...  # can be public in frontend code
STRIPE_WEBHOOK_SECRET=whsec_...

# Other providers (if used)
PAYPAL_CLIENT_ID=
PAYPAL_CLIENT_SECRET=
TAP_SECRET_KEY=
```

---

## SPECIFIC DO NOTS

- Never store raw card numbers or CVVs
- Never log card details even briefly
- Never trust success URLs as payment confirmation — use webhooks
- Never skip webhook signature verification
- Never use floats for monetary amounts — always integers in smallest unit
- Never hardcode amounts or prices in the frontend
- Never expose secret keys
- Never implement your own payment form that captures card data
- Never charge without explicit user consent for each charge (subscriptions must be clearly disclosed)
- Never skip refund handling — it's legally required in many jurisdictions

---

## SESSION COMPLETION CHECKLIST — PAYMENTS SPECIFIC

- [ ] Webhook endpoint exists and verifies signatures
- [ ] All amounts stored as integers (cents/halalas)
- [ ] Payment records stored in database with full audit trail
- [ ] Idempotency handled in webhooks
- [ ] Test cards work in test mode
- [ ] Webhook forwarding set up for local dev
- [ ] Secret keys in environment variables
- [ ] Frontend never sees secret keys
- [ ] Error messages are user-friendly
- [ ] Loading states prevent double-submissions

---

*This addon is part of Forge by Sultan Al-Muqimi / NQTH LLC.*
