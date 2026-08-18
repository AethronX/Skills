---
name: payments-and-billing
description: Implements payment processing, subscriptions, and billing correctly. Use when integrating a payment provider (Stripe, etc.), building checkout flows, handling subscriptions or usage-based billing, processing webhooks for payment events, or handling refunds and disputes.
---

# Payments and Billing

## Overview

Payment code has the least tolerance for "close enough" of anything you'll build: a bug here either charges someone wrong, double-charges them, or silently fails to grant what they paid for. The unifying principle is **the payment provider is the source of truth, not your UI** — every decision below follows from treating client-side state, redirect callbacks, and even your own database as provisional until confirmed by a provider event.

## When to Use

- Integrating a payment provider (Stripe, Paddle, Braintree, etc.)
- Building checkout, cart, or pricing pages
- Implementing subscriptions, trials, or usage-based billing
- Handling payment webhooks
- Processing refunds, disputes, or chargebacks
- Reviewing existing payment code for correctness

## Core Rules

### Never Trust the Client for Price or Amount

```typescript
// BAD: client sends the amount to charge
app.post('/api/checkout', async (req, res) => {
  await charge(req.body.amount); // a modified request charges whatever the client says
});

// GOOD: server looks up the price from its own source of truth
app.post('/api/checkout', async (req, res) => {
  const { priceId } = req.body; // client sends an identifier, not an amount
  const price = await db.prices.findUnique({ where: { id: priceId } });
  if (!price) return res.status(404).json({ error: 'unknown price' });
  await charge(price.amountCents, price.currency);
});
```

The client may choose *what* to buy; it never gets to say *how much that costs*.

### Store Money as Integers in the Smallest Unit

```typescript
// BAD: floating point money — 19.99 + 0.01 !== 20.00 in IEEE 754
const total = 19.99 + 0.01;

// GOOD: integer cents (or the provider's smallest unit)
const totalCents = 1999 + 1; // 2000
```

Never store or compute prices as floats. Use integer minor units (cents) or a fixed-point decimal library. Convert to display format only at render time.

### The Webhook Is the Source of Truth, Not the Redirect

A user landing on `/checkout/success` proves they were *redirected there* — it does not prove payment succeeded. Browsers close, redirects fail, users bookmark and revisit stale URLs.

```typescript
// BAD: granting access on the success-page visit
app.get('/checkout/success', async (req, res) => {
  await grantAccess(req.query.userId); // no proof payment actually completed
  res.render('success');
});

// GOOD: grant access only from the verified webhook event
app.post('/webhooks/stripe', async (req, res) => {
  const event = stripe.webhooks.constructEvent(
    req.body, req.headers['stripe-signature'], process.env.STRIPE_WEBHOOK_SECRET
  );

  if (event.type === 'checkout.session.completed') {
    await grantAccess(event.data.object.client_reference_id);
  }
  res.json({ received: true });
});

// The success page just shows a "processing" state and polls/waits for the
// webhook-driven state change — it never grants anything itself.
```

**Always verify the webhook signature.** An unauthenticated `/webhooks/*` endpoint that trusts the payload is an open door to fabricate "payment succeeded" events.

### Webhooks Arrive At-Least-Once — Processing Must Be Idempotent

The same event can be delivered more than once (retries after a timeout, provider-side redelivery). Dedupe by the event's unique ID before acting, using the same atomic-claim pattern as any idempotency key (see the `api-and-interface-design` skill):

```typescript
app.post('/webhooks/stripe', async (req, res) => {
  const event = stripe.webhooks.constructEvent(/* ... */);

  try {
    await db.processedEvents.insert({ id: event.id }); // unique constraint on id
  } catch (e) {
    if (isUniqueViolation(e)) return res.json({ received: true }); // already handled
    throw e;
  }

  await handleEvent(event);
  res.json({ received: true });
});
```

### Never Touch Raw Card Data

Use the provider's hosted checkout, Elements, or a similar tokenizing component — the card number should never pass through your server as plaintext. Handling raw PANs pulls you into PCI-DSS scope you almost certainly don't want. This is a specific case of the general rule in `security-and-hardening`: never handle sensitive data your system doesn't need to see.

## Subscriptions and Recurring Billing

### Events Worth Handling (Provider-Agnostic Shape)

| Event | Action |
|---|---|
| Checkout/subscription created | Grant access, store the subscription ID against the user |
| Invoice payment succeeded | Extend access period, reset any "payment failed" flags |
| Invoice payment failed | Start dunning (retry + notify); don't revoke access on the first failure |
| Subscription updated (plan change, proration) | Sync entitlements to the new plan immediately |
| Subscription deleted/canceled | Revoke access — but respect the paid-through date, don't cut off mid-period |
| Dispute/chargeback created | Flag the account; freeze rather than silently continue service |

### Dunning (Failed Payment Retries)

Don't revoke access on the first declined charge — cards fail transiently (expired, temporary hold, bank flagged as suspicious). Use the provider's built-in retry schedule where available, notify the user at each attempt, and only downgrade/cancel after the retry window exhausts (commonly 3–4 attempts over 1–2 weeks).

### Proration and Plan Changes

Let the provider compute proration (Stripe, etc. do this correctly out of the box) rather than hand-rolling prorated amounts — the edge cases (mid-cycle upgrade, downgrade, annual-to-monthly) are easy to get subtly wrong and directly affect what someone is charged.

## Testing

- Use the provider's test/sandbox mode and documented test card numbers — never test against real cards, even your own.
- Forward webhooks to localhost during development with the provider's CLI (e.g. `stripe listen --forward-to localhost:3000/api/webhooks`) so the webhook path is actually exercised, not just the redirect path.
- Test the failure paths deliberately: declined card, expired card, webhook signature mismatch, duplicate webhook delivery, subscription cancellation mid-period.

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "The success page redirect is good enough" | Redirects aren't proof of payment. Only the provider's server-to-server webhook is. |
| "We'll just use floats, it's close enough" | Floating-point rounding compounds across a ledger. Integer minor units are not optional. |
| "Our webhook endpoint doesn't need signature verification, it's obscure" | An obscure URL is not authentication. Anyone who finds it can forge "paid" events without verification. |
| "Duplicate webhook deliveries are rare" | They spike exactly when your endpoint is slow or the network is degraded — the same moment double-processing is most damaging. |
| "We'll cut off access immediately on payment failure" | Legitimate cards fail transiently. Immediate cutoff creates support tickets for money you'll still collect on retry. |

## Red Flags

- Amount or price computed/trusted from client input
- Money stored or computed as a floating-point number
- Access or fulfillment granted from a redirect/success page rather than a verified webhook
- Webhook endpoint with no signature verification
- Webhook handler with no idempotency/dedupe — reprocesses the same event on redelivery
- Raw card numbers ever touching your own server or logs
- Immediate access revocation on the first failed payment attempt
- Hand-rolled proration math

## Verification

- [ ] All prices/amounts are looked up server-side from a trusted source, never trusted from the client
- [ ] Money is represented as integer minor units (or fixed-point decimal), never float
- [ ] Access/fulfillment is granted only from a verified, signature-checked webhook event
- [ ] Webhook processing is idempotent (deduped by event ID) against redelivery
- [ ] No raw card data ever reaches your server (hosted checkout/Elements/tokenization used)
- [ ] Failed payments trigger dunning, not immediate revocation
- [ ] Subscription cancellation respects the paid-through period
- [ ] Test mode and sandbox webhooks were exercised for both success and failure paths
