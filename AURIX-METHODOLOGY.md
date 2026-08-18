# AURIX WORK — Skills Orchestration Methodology

This document describes how to select, combine, and sequence the skills in this repository when building a production digital product for AURIX (premium websites, e-commerce stores, SaaS apps, dashboards, AI-powered applications, automation systems, internal tools). It is a methodology, not a skill — it doesn't trigger on its own; an agent (or a human) follows it when deciding *which* of the skills below to load and *in what order*, for a substantial build.

## Core Objective

Every AURIX project is judged on ten dimensions, each targeted at 10/10: **Design, UX, Responsiveness, Code Quality, Performance, Security, SEO, Accessibility, Functionality, Maintainability.**

These are targets, not permission to fabricate results. If a category genuinely cannot reach the target within the project's real constraints, say so explicitly and name the limitation — don't claim a score that wasn't verified.

## Skill Selection Rules

1. **Discover before building.** Check the skill index below (or `README.md`) for what's actually available. Never assume a skill exists that isn't listed there.
2. **Select only what the task needs.** Loading every skill for a one-page brief adds context without adding value. Match skills to the modes actually in play (see Mode Activation Map).
3. **Combine specialized skills rather than one generalist pass.** A checkout page needs `ecommerce-storefront` *and* `payments-and-billing` *and* `security-and-hardening` — none of them alone covers it.
4. **When skills overlap, prioritize in this order:** official/trusted implementation → highly specialized skill → production-grade guidance → actively maintained → AURIX-specific requirement → general-purpose fallback. Don't blindly merge conflicting instructions from two overlapping skills — pick the more specific one for the situation at hand.

## The AURIX Development Pipeline

Work moves through these stages. Don't skip a stage just because the app already "seems to work" — most of the stages below exist specifically to catch what "seems to work" misses.

| Stage | What happens | Primary skills |
|---|---|---|
| **Discover / Analyze / Plan** | Requirements, users, business objective, pages, features, integrations, auth/payment/database needs | *(scoping — no dedicated skill; this is where you decide which modes below apply)* |
| **Architect** | API contracts, data model, module boundaries | `api-and-interface-design`, `database-design` |
| **Design** | Typography, spacing, color, component hierarchy, states, motion | `frontend-ui-engineering`, `animation` |
| **Implement** | Components, pages, business logic | `frontend-ui-engineering`, `organizing-project-files`, plus mode-specific skills (below) |
| **Integrate** | Third-party APIs, webhooks, payment/AI providers | `api-integration`, `api-and-interface-design` |
| **Test** | Unit/integration/e2e, real-browser verification | `writing-tests`, `browser-testing-with-devtools` |
| **Security Audit** | Auth, input validation, secrets, injection, webhook security | `security-and-hardening` |
| **Performance Audit** | Core Web Vitals, bundle size, N+1 queries, caching | `performance-optimization` |
| **SEO Audit** | Metadata, structured data, sitemap, crawlability, AEO | `seo-and-discoverability` |
| **Accessibility Audit** | Keyboard nav, contrast, ARIA, screen readers | `frontend-ui-engineering` (WCAG section + checklist) |
| **Final QA** | Code review, definition-of-done | `reviewing-code`, `shipping-and-launch` (pre-launch checklist) |
| **Deploy** | CI/CD, staged rollout, rollback plan | `ci-cd-and-automation`, `shipping-and-launch`, `using-cli-tools` |
| **Post-Deploy** | Logging, metrics, tracing, alerting | `observability-and-instrumentation` |

`writing-commits` applies throughout, for every commit and PR along the way — it's not a pipeline stage, it's a constant.

## Mode Activation Map

Activate a mode's skills only when the project actually needs that mode — this is what keeps "select only what's needed" from becoming "load everything."

| Mode | Trigger | Activate |
|---|---|---|
| **E-commerce** | The project sells something | `ecommerce-storefront` + `payments-and-billing` + `database-design` + `seo-and-discoverability` |
| **AI Application** | The project includes an LLM feature, agent, or voice AI | `ai-integration-and-agents` + `security-and-hardening` (LLM section) + `observability-and-instrumentation` — never activate this mode just because AI is *possible*; it must solve a real problem in the brief |
| **API & Integration** | The project calls external services and/or exposes its own API | `api-integration` for consuming, `api-and-interface-design` for exposing — both, if it does both |
| **Security** | Always, before completion — not optional | `security-and-hardening` |
| **Performance** | Always, before completion | `performance-optimization` + `browser-testing-with-devtools` for real measurement |
| **SEO** | Any public-facing page | `seo-and-discoverability` — implemented during development, not bolted on after |
| **Accessibility** | Always, for any UI | `frontend-ui-engineering` |
| **Testing** | Always | `writing-tests` + `browser-testing-with-devtools` |
| **Code Review** | Before declaring anything complete | `reviewing-code` |
| **Mobile-first** | Always — mobile is not an afterthought | `frontend-ui-engineering` (responsive design section) |

## Non-Negotiable Constraints

- **No fake functionality.** Never fabricate analytics, orders, customers, payments, API responses, AI responses, or database records to make something look finished. If real functionality isn't wired up yet, label demo data as demo data, visibly, not as if it were production.
- **No overengineering.** No unnecessary dependencies, microservices, abstractions, state-management layers, animations, AI, or infrastructure. Use the simplest architecture that reliably satisfies the actual requirement — this is the same principle `performance-optimization`'s "don't optimize before you have evidence" and `api-and-interface-design`'s addition-over-modification guidance both apply in their own domains.
- **Existing-project protection.** When modifying a project that already has working code: inspect it first, identify what already works, and change only what's necessary. Preserve existing integrations, environment variables, database structure, auth, and deployment config unless there's a clear, stated reason to change them. Don't rewrite something just to make the code look different.

## Final Audit Report Format

Before declaring a substantial task complete, report against this structure — every section grounded in what was actually done, not asserted:

```
COMPLETED       — what was successfully implemented
SKILLS USED     — which skills were activated and why
TESTS           — what was actually tested (and how)
SECURITY        — what was checked
PERFORMANCE     — what was checked
SEO             — what was checked
ACCESSIBILITY   — what was checked
REMAINING ISSUES — anything that still needs attention, stated plainly
```

## Master Rule

Optimize for *"how can I deliver the highest-quality production system with the smallest necessary complexity"* — not for *"how quickly can I generate code."* Think before coding. Inspect before modifying. Test before declaring success. Never claim something is complete unless it has actually been implemented and verified.
