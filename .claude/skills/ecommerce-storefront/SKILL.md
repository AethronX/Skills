---
name: ecommerce-storefront
description: Builds e-commerce storefronts — product catalogs, cart, checkout, search/filtering, and conversion-focused product pages. Use when building or reviewing a product listing page, product detail page, cart, checkout flow, or storefront search. For payment/webhook correctness see payments-and-billing; for the underlying schema see database-design; for page discoverability see seo-and-discoverability — use together.
---

# E-Commerce Storefront

## Overview

A storefront has its own failure modes on top of everything general frontend/backend work already has to get right: selling something you don't have, charging the wrong price, or losing a cart mid-checkout costs real money and trust in a way a typical CRUD feature doesn't. This skill covers the storefront-specific concerns — catalog modeling, inventory correctness, search, and checkout conversion — and points to the general skills that cover the rest.

## When to Use

- Building or modifying a product listing page (PLP) or product detail page (PDP)
- Designing the cart, checkout flow, or order confirmation
- Implementing product search, filtering, or faceting
- Modeling products, variants, and inventory
- Reviewing a storefront for conversion or correctness issues

## Product Catalog Modeling

```
Product              → the sellable concept ("Classic T-Shirt")
  └── Variants        → the actual purchasable SKUs (size × color combinations)
        ├── price      → variants can have different prices (a rarely-obvious bug source
        │                if the UI assumes one price per product)
        ├── sku
        ├── inventory
        └── images      → often per-variant (color-specific photos)
```

- **Model variants as first-class records with their own price and inventory**, not as attributes bolted onto the product — a product-level price field breaks the moment two variants cost different amounts.
- **Options vs. variants**: "Size" and "Color" are *options*; "Medium / Red" is the *variant* that combination resolves to. Not every combination of options needs to exist (don't assume a full cartesian product).
- See the `database-design` skill for the underlying schema/indexing decisions once the shape above is settled.

## Inventory Correctness

The single most damaging storefront bug is **overselling**: two customers buy the last unit because both checkouts read "1 in stock" before either write completed.

```typescript
// BAD: check-then-act race condition
const product = await db.products.findUnique({ where: { id } });
if (product.stock > 0) {
  await db.products.update({ where: { id }, data: { stock: product.stock - 1 } });
  // Two concurrent requests can both pass the check before either decrements.
}

// GOOD: atomic conditional decrement — the database enforces the invariant
const result = await db.products.updateMany({
  where: { id, stock: { gt: 0 } },
  data: { stock: { decrement: 1 } },
});
if (result.count === 0) {
  throw new OutOfStockError();
}
```

- **Decrement inventory atomically**, conditioned on availability, in the same operation — never read-then-write across two round trips.
- **Reserve inventory at checkout start, not just at payment success**, for anything with meaningfully limited stock — otherwise a slow checkout (entering card details) can lose a race to someone who started later. Release the reservation on cart abandonment/timeout.
- **Decide your oversell policy up front** for edge cases (backorder allowed vs. hard block) rather than discovering it in production when a race condition already sold the last unit twice.
- **Low-stock display thresholds should read from the same source of truth** used to block checkout — a "Only 2 left!" banner that doesn't match the actual atomic check just creates confused support tickets.

## Cart

- **Persist the cart** across sessions for both guest and logged-in users; merge a guest cart into the account cart on login rather than discarding one.
- **Never trust cart prices from client state at checkout.** Re-fetch current prices server-side when the order is placed — this is the same rule as the `payments-and-billing` skill's "never trust the client for price," applied to the moment between "added to cart" and "paid," during which a price or promotion may have changed.
- **Show the user when something changed** (price update, item went out of stock, promotion expired) rather than silently adjusting the total at checkout — silent changes are the single fastest way to make a customer distrust checkout.

## Search and Filtering

- **Faceted filtering** (price range, size, color, brand, rating) needs each facet's available options computed from the *currently filtered* result set, not the full catalog — showing a color filter option with zero matching results after other filters are applied is a common, confusing bug.
- **Typo tolerance and synonym handling** matter more here than in most search: a shopper who mistypes a brand name and gets zero results leaves; a general-purpose text index (Postgres full-text, Elasticsearch/Algolia/Typesense) with fuzzy matching is usually worth the setup cost above a few hundred SKUs.
- **Paginate or virtualize large result sets** — this is the `performance-optimization` skill's unbounded-query and long-list guidance, and it applies especially hard to a PLP that can return thousands of matches.

## Product Page Conversion

- **Price and primary call-to-action visible without scrolling** on the most common viewport sizes — this is the single highest-leverage layout decision on a PDP.
- **Real product images with dimensions set** (no layout shift as images load) — ties to `performance-optimization`'s CLS guidance, and it's disproportionately visible on an image-heavy page.
- **Show trust signals near the decision point**: return policy, shipping estimate, stock status, reviews — next to the buy button, not buried in a separate tab nobody opens.
- **Reviews/ratings should carry `Product`/`AggregateRating` structured data** — see the `seo-and-discoverability` skill; this is one of the highest-value structured-data additions for e-commerce specifically (star ratings in search results measurably affect click-through).
- **Don't block the "Add to Cart" action on anything non-essential** (a newsletter modal, an unnecessary account-creation step) — every extra required step before purchase is a measurable drop in conversion.

## Checkout

- **Offer guest checkout.** Forcing account creation before purchase is one of the most well-documented conversion killers in e-commerce; let people buy first, offer account creation after.
- **Minimize required form fields** and use address autocomplete where available — every additional required field on a checkout form measurably increases abandonment.
- **Re-validate everything server-side at the moment of payment**: price, stock, promotion eligibility, shipping cost. The cart and checkout UI are conveniences for the user, not the source of truth for what gets charged (see `payments-and-billing`).
- **Recover abandoned carts deliberately** (email reminder, saved cart on return) rather than only relying on the user coming back on their own.

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "Stock checks are fast, races are rare" | Races correlate with your busiest, most valuable traffic (a flash sale, a viral product) — the exact moment overselling costs the most in refunds and trust. |
| "We'll just trust the cart price at checkout" | Prices and promotions change between add-to-cart and purchase. Re-validate server-side, every time. |
| "Account creation before checkout captures more leads" | It also loses more sales than it captures, for most storefronts. Offer it after purchase instead. |
| "Search doesn't need typo tolerance, people can spell" | They can't, reliably, on mobile keyboards, for brand names they've only heard once. Zero results reads as "this store doesn't have it." |
| "One price field on the product is simpler" | It's simpler until the first product needs two variants at different prices, and then it's a migration. |

## Red Flags

- Inventory checked and decremented in two separate operations (read-then-write race)
- Product-level (not variant-level) price field on a catalog with variants
- Checkout that trusts the client-submitted cart total
- Facet filters showing options with zero results after other filters are applied
- No guest checkout option
- Product images without explicit dimensions (layout shift on load)
- No structured data on product/review content

## Verification

- [ ] Inventory decrements are atomic and conditional on availability (no check-then-act race)
- [ ] Variants carry their own price, SKU, and inventory
- [ ] Cart/checkout totals are recomputed server-side from current prices at the moment of payment
- [ ] Out-of-stock or price-changed items are surfaced to the user, not silently adjusted
- [ ] Search tolerates common typos and returns facets scoped to the current filtered result set
- [ ] PLP/PDP images have explicit dimensions and are optimized (see `performance-optimization`)
- [ ] Guest checkout is available; required form fields are minimized
- [ ] Product/review content carries `Product`/`AggregateRating` structured data (see `seo-and-discoverability`)
