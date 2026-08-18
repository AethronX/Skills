---
name: api-integration
description: Integrates with third-party APIs and webhooks reliably. Use when calling an external API/SDK, handling rate limits or retries, receiving webhooks from a third-party service, or wrapping an external provider behind your own interface. For designing APIs your own system exposes, see api-and-interface-design instead — use both together when a feature does both.
---

# API Integration

## Overview

Designing an API (covered by the `api-and-interface-design` skill) and *consuming* someone else's are different disciplines. When you're the caller, you don't control uptime, latency, rate limits, or breaking changes — the job is to build resilience around a dependency you can't fix when it misbehaves, and to keep that dependency from leaking into every corner of your codebase.

## When to Use

- Calling a third-party REST/GraphQL API or SDK
- Handling that provider's rate limits, retries, or outages
- Receiving webhooks from an external service
- Wrapping a third-party SDK behind your own interface
- Choosing an authentication flow for a service integration (API key vs. OAuth)
- Reviewing integration code for resilience or correctness

## Isolate the Dependency Behind Your Own Interface

```typescript
// BAD: the third-party SDK's shapes leak throughout the codebase
import Stripe from 'stripe';
async function chargeCustomer(stripe: Stripe, customerId: string) { /* ... */ }
// Every caller now depends on Stripe's types directly.

// GOOD: define your own interface; the adapter is the only place that imports the SDK
interface PaymentProvider {
  charge(customerId: string, amountCents: number): Promise<ChargeResult>;
}

class StripePaymentProvider implements PaymentProvider {
  async charge(customerId: string, amountCents: number): Promise<ChargeResult> {
    const result = await this.stripe.paymentIntents.create({ /* ... */ });
    return { id: result.id, status: mapStripeStatus(result.status) };
  }
}
```

Swapping or adding a second provider later touches one adapter, not every call site. This also gives you a single place to add retries, logging, and error mapping consistently.

## Resilience Patterns

### Always Set a Timeout

An external call with no timeout can hang your request indefinitely on someone else's outage. Every outbound call needs an explicit timeout shorter than your own service's SLA allows.

### Retry With Backoff and Jitter — Only for Safe Operations

```typescript
async function withRetry<T>(fn: () => Promise<T>, maxAttempts = 3): Promise<T> {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (err) {
      if (attempt === maxAttempts || !isRetryable(err)) throw err;
      const backoff = 2 ** attempt * 100;
      const jitter = Math.random() * 100;
      await sleep(backoff + jitter);
    }
  }
  throw new Error('unreachable');
}

function isRetryable(err: unknown): boolean {
  // Retry timeouts and 5xx; never blindly retry a 4xx (it will fail identically)
  // or a non-idempotent write unless it carries an idempotency key.
  return isTimeout(err) || (isHttpError(err) && err.status >= 500);
}
```

- **Only retry idempotent operations** (GETs, or writes made idempotent via a key — see `api-and-interface-design`'s idempotency-key guidance) — retrying a plain `POST /charge` on a timeout can duplicate the charge; you don't know if the first attempt actually succeeded on the provider's side.
- **Jitter prevents a retry storm**: without it, many clients retrying on the same fixed schedule after a shared failure all hit the recovering service at once.
- **Never retry a 4xx** (except 429) — the request is malformed or unauthorized, and retrying it identically fails identically.

### Circuit Breaker

When a dependency is clearly down, stop calling it for a cooldown window instead of letting every request pay the same timeout — this protects your own service's latency and the failing dependency from a retry storm while it recovers.

```
Circuit states:
CLOSED  (normal) → too many recent failures → OPEN (fail fast, no calls sent)
OPEN → after cooldown → HALF-OPEN (allow one test call)
HALF-OPEN → success → CLOSED   |   HALF-OPEN → failure → OPEN again
```

### Respect Rate Limits

- Read the provider's rate-limit response headers (`Retry-After`, `X-RateLimit-Remaining`) and back off accordingly rather than guessing.
- On a `429`, wait for the duration the provider tells you, not a fixed short retry — hammering through a rate limit gets you blocked harder or IP-banned.
- For high-volume integrations, queue and throttle client-side to stay under the limit proactively, rather than reactively handling 429s as the normal case.

## Authentication Patterns

| Pattern | Use for | Key rule |
|---|---|---|
| API key / secret | Server-to-server, no per-user delegation | Server-side only — never ship a secret key to a browser or mobile client |
| OAuth (authorization code + PKCE) | Acting on behalf of a specific user, with their consent | Store refresh tokens encrypted; handle expiry/refresh transparently to the caller |
| Signed webhooks (HMAC) | Receiving events from the provider | Verify the signature before trusting the payload — see below |

Never put a server-side secret in client-side code or a mobile app bundle — anything shipped to a browser or app is extractable. If a browser needs to call the provider directly, use a scoped, short-lived, publishable/public key designed for that (most major providers offer one) — not the private key.

## Receiving Webhooks

This mirrors the `payments-and-billing` skill's webhook guidance, generalized to any provider:

- **Verify the signature** on every incoming webhook before acting on it — an unauthenticated webhook endpoint lets anyone forge events.
- **Acknowledge fast, process async.** Return `200` as soon as the event is durably queued (or the minimal synchronous work is done); do the heavy processing in a background job. Providers typically time out and retry a slow handler, which then causes duplicate delivery on top of being slow.
- **Deduplicate by the event's unique ID** — webhooks are delivered at-least-once; treat redelivery as normal, not exceptional.
- **Don't assume delivery order.** Events can arrive out of order under retry; design handlers to be correct regardless of sequence (check current state, not "this must be the next expected event").

## Treat Responses as Untrusted Data

A third-party API response — even from a provider you trust — can return unexpected shapes, error conditions disguised as 200s, or (per `security-and-hardening`'s LLM guidance, if the provider is AI-backed) content designed to manipulate downstream logic. Validate the response shape before using it, the same as any external input:

```typescript
const ProviderResponseSchema = z.object({
  status: z.enum(['success', 'pending', 'failed']),
  data: z.object({ id: z.string(), amount: z.number() }),
});

const parsed = ProviderResponseSchema.safeParse(await response.json());
if (!parsed.success) {
  throw new UpstreamContractError('unexpected response shape from provider');
}
```

## Testing

- Mock the third party in unit/integration tests — don't call the real API in CI.
- Use the provider's sandbox/test mode for end-to-end tests where one exists.
- Test your resilience code deliberately: simulate a timeout, a 500, a 429, and a malformed response, and confirm the retry/circuit-breaker/error-mapping logic actually does what it's supposed to.
- If the provider offers contract tests or a schema, use them to catch breaking changes on their side before production does.

## Observability

Every external dependency should show up in your telemetry as its own thing, not folded into generic request logs — see the `observability-and-instrumentation` skill's RED-metrics guidance applied to "external dependency" as a label:

- Rate, error rate, and latency per external provider/endpoint
- Retry counts and circuit-breaker state transitions
- Alert on a dependency's error rate independent of your own service's error rate — "our error rate is fine" can hide "this one provider is failing 40% of calls, masked by everything else succeeding."

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "We'll just call the SDK directly everywhere" | Every call site now depends on that vendor's types and error shapes. Swapping or adding a provider later touches the whole codebase. |
| "Retrying is always safe, it's just a network blip" | Retrying a non-idempotent write on a timeout can duplicate it — you don't know if the original request already succeeded upstream. |
| "We'll handle 429s by retrying immediately" | That's how you get rate-limited harder or IP-banned. Respect `Retry-After`. |
| "The webhook handler can just do the work inline" | Slow synchronous processing causes the provider to time out and redeliver — now you're handling the same event twice, slowly. |
| "It's a trusted provider, we don't need to validate the response" | Trusted providers still change their API, have bugs, and return malformed data during their own incidents. Validate the shape. |

## Red Flags

- Third-party SDK types/calls scattered directly across the codebase instead of behind an adapter
- Outbound calls with no timeout
- Retry logic applied uniformly to both idempotent and non-idempotent operations
- No backoff/jitter — fixed-interval retries that can synchronize into a retry storm
- Webhook endpoint with no signature verification
- Webhook handler doing heavy synchronous work before acknowledging
- No dedup on webhook event ID
- Provider API keys or secrets present in client-side/browser code
- Third-party responses used without validating their shape

## Verification

- [ ] The third-party SDK/client is wrapped behind an interface your own code depends on, not called directly everywhere
- [ ] Every outbound call has an explicit timeout
- [ ] Retries are limited to idempotent operations (or operations carrying an idempotency key), with backoff and jitter
- [ ] Rate-limit responses are respected (`Retry-After` honored, proactive throttling for high-volume calls)
- [ ] A circuit breaker (or equivalent) stops hammering a dependency that's clearly down
- [ ] Secrets used for the integration are server-side only
- [ ] Incoming webhooks verify their signature before being trusted
- [ ] Webhook handlers acknowledge fast and process heavy work asynchronously
- [ ] Webhook processing is deduplicated by event ID and correct regardless of delivery order
- [ ] Third-party response shapes are validated before use
- [ ] The integration has its own rate/error/latency metrics, alertable independently of overall service health
