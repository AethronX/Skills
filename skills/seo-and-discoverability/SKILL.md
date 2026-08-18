---
name: seo-and-discoverability
description: Makes web pages discoverable by search engines and AI answer engines. Use when building pages meant to be found (marketing pages, blog posts, product pages, docs), when adding metadata/structured data, when setting up a sitemap or robots.txt, or when content needs to be citable by AI assistants (ChatGPT, Perplexity, AI Overviews).
---

# SEO and Discoverability

## Overview

Discoverability in 2030 means two audiences, not one: search engine crawlers (Google, Bing) and AI answer engines that read, summarize, and cite pages (ChatGPT browsing, Perplexity, Google AI Overviews, Claude with web access). Both need the same foundation — crawlable, well-structured, fast, unambiguous content — but the second audience cares more about extractability than keyword density. Optimize for being *correctly understood and cited*, not just *ranked*.

## When to Use

- Building any page meant to be found organically: marketing/landing pages, blog posts, docs, product pages
- Adding or auditing `<meta>` tags, Open Graph data, or structured data
- Setting up `sitemap.xml`, `robots.txt`, or canonical URLs
- Writing content that should be quotable/citable by AI assistants
- Reviewing a page that ranks poorly or isn't appearing in AI-generated answers

**When NOT to use:** Authenticated app views, internal dashboards, or anything behind a login wall — these should generally be `noindex` and don't need SEO investment.

## Technical Foundation (Table Stakes)

### Crawlability

Both search bots and most AI-answer crawlers either don't execute JavaScript or execute it unreliably. Content that only appears after a client-side fetch or an interaction (accordion, "load more", client-only render) may be invisible to them.

- **Server-render or statically generate** any page meant to be indexed (Next.js: `generateStaticParams`/SSR, not client-only `useEffect` data fetching for primary content).
- **Don't gate primary content behind interaction.** An FAQ answer hidden until a click should still be present in the initial HTML (visually collapsed via CSS, not absent from the DOM).
- **`robots.txt`** should allow the crawlers you want and explicitly block none you need — check it doesn't accidentally block CSS/JS the renderer needs to evaluate the page as "mobile-friendly."

### Metadata

```typescript
// Next.js App Router: generateMetadata per route
export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const product = await getProduct(params.id);
  return {
    title: `${product.name} | Acme Store`,
    description: product.shortDescription, // 150–160 chars, unique per page
    alternates: { canonical: `https://acme.com/products/${product.id}` },
    openGraph: {
      title: product.name,
      description: product.shortDescription,
      images: [{ url: product.heroImage, width: 1200, height: 630 }],
      type: 'website',
    },
    twitter: { card: 'summary_large_image' },
  };
}
```

- **Every indexable page has a unique `<title>` and `description`.** Duplicate or missing metadata is the single most common SEO defect.
- **Canonical URLs** on any content reachable through multiple paths (query params, trailing slash, `www` vs. apex) to avoid duplicate-content dilution.
- **Open Graph + Twitter Card** tags for every page that might be shared — without them, social previews degrade to a bare link.

### Structured Data (schema.org / JSON-LD)

Structured data tells crawlers and AI systems *what a page is*, unambiguously, instead of leaving them to infer it from prose:

```typescript
// Embed as a <script type="application/ld+json"> in the page
const productSchema = {
  '@context': 'https://schema.org',
  '@type': 'Product',
  name: product.name,
  description: product.shortDescription,
  offers: {
    '@type': 'Offer',
    price: product.price,
    priceCurrency: 'USD',
    availability: product.inStock
      ? 'https://schema.org/InStock'
      : 'https://schema.org/OutOfStock',
  },
  aggregateRating: product.reviewCount > 0 ? {
    '@type': 'AggregateRating',
    ratingValue: product.avgRating,
    reviewCount: product.reviewCount,
  } : undefined,
};
```

Common types worth adding where they apply: `Organization`, `Product`, `Article`/`BlogPosting`, `FAQPage`, `BreadcrumbList`, `HowTo`, `LocalBusiness`. Validate with Google's Rich Results Test before shipping — malformed structured data is worse than none (it can trigger manual penalties).

### Sitemap and Indexing Signals

```xml
<!-- sitemap.xml — keep it current, submit via Search Console -->
<url>
  <loc>https://acme.com/products/widget</loc>
  <lastmod>2026-08-01</lastmod>
  <changefreq>weekly</changefreq>
</url>
```

- Generate the sitemap from the same data source as the routes (don't hand-maintain a static file that drifts from reality).
- `noindex` anything that shouldn't rank: internal tools, thank-you/checkout-confirmation pages, paginated duplicate content, staging environments.
- One canonical domain (redirect `www` ↔ apex consistently, HTTP → HTTPS always).

## Core Web Vitals

Search ranking and AI-crawler patience both correlate with load performance. This is the `performance-optimization` skill's territory — the specific overlap for SEO: **LCP, INP, and CLS are direct or indirect ranking factors**, so a slow page is an SEO defect, not just a UX one. Don't duplicate that skill's checklist here; run it.

## Optimizing for AI Answer Engines (AEO)

AI assistants that browse or have been trained on the web tend to reward the same things a rushed human skimmer would: a clear, early, literal answer — not clever copy.

- **Answer the question in the first 1–2 sentences**, then elaborate. Don't bury the direct answer under three paragraphs of scene-setting — many extraction pipelines weight the opening content most heavily.
- **Use real headings that match the questions users ask.** An `<h2>` that literally states "How much does shipping cost?" gets extracted more reliably than "Shipping Details."
- **FAQ sections with `FAQPage` structured data** are one of the highest-leverage additions for AI citation — they're explicitly structured as question/answer pairs, which is exactly the shape these systems extract.
- **State facts as facts, not as marketing.** "Free shipping on orders over $50" is extractable; "Enjoy blazing-fast delivery that'll make you smile" is not.
- **Keep a single canonical version of any fact** (price, spec, policy) on the site. Contradicting numbers on two pages get either averaged wrong or cause the AI system to distrust the source.
- **Consider an `llms.txt`** at the site root — an emerging, not-yet-universal convention (proposed 2024) for pointing AI crawlers at your most important, cleanly-formatted content (similar spirit to `robots.txt`/`sitemap.xml`). Treat it as a cheap, low-risk addition, not a guaranteed lever — support among AI crawlers is inconsistent and evolving.
- **Attribution helps you as much as the reader.** Citing sources and dates on factual claims makes a page more likely to be quoted with your brand attached, rather than paraphrased anonymously.

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "SEO is just keywords" | Keyword stuffing is actively penalized. Structure, speed, and unambiguous facts matter more now. |
| "We'll add metadata later" | Pages get indexed with whatever metadata exists at crawl time; "later" often means after the bad version is cached. |
| "AI crawlers will figure it out" | Extraction pipelines reward explicit structure (headings, FAQ schema, early answers). Ambiguous prose gets misquoted or skipped. |
| "Client-side rendering is fine, Google runs JS now" | Google's JS rendering is a second, delayed pass — and most AI-answer crawlers don't run JS at all. SSR/SSG for anything meant to be found. |
| "One page, three URLs, no big deal" | Duplicate-content variants (with/without trailing slash, `?ref=`, `www` vs. apex) split ranking signal across copies. Canonicalize. |

## Red Flags

- Missing or duplicate `<title>`/`<meta description>` across pages
- Primary content only rendered client-side or behind an interaction
- No canonical URL on pages reachable via multiple paths
- No structured data on content types that clearly map to a schema.org type (products, articles, FAQs)
- Sitemap that's stale, hand-maintained, or missing entirely
- Marketing copy where a direct factual answer should be
- Contradicting facts (price, specs, policy) across different pages of the same site

## Verification

- [ ] Every indexable page has a unique title and meta description
- [ ] Primary content is present in server-rendered/static HTML, not client-fetch-only
- [ ] Canonical URL set on every page reachable through more than one path
- [ ] Structured data present for applicable content types and validated (no schema errors)
- [ ] Sitemap generated from live route data, submitted to Search Console
- [ ] `robots.txt` doesn't block assets the renderer needs, and does `noindex` non-public pages
- [ ] Core Web Vitals pass "Good" thresholds (see `performance-optimization`)
- [ ] Key questions are answered directly within the first 1–2 sentences under their heading
- [ ] FAQ-shaped content uses `FAQPage` structured data
