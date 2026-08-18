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

## Session Initialization

At the start of every substantial AURIX task (not for small, single-file edits — don't reread unchanged docs on every trivial action):

1. Read `CLAUDE.md`, `AGENTS.md`, and this file (already in context if this repo is open; re-read only if something changed or context was reset).
2. Inspect the skills actually available (the list in `README.md` / `skills/` — never assume one exists that isn't there).
3. Determine which skills the task requires, using the Automatic Skill Orchestration table below.
4. Form an internal execution plan before modifying any code.

## Automatic Skill Orchestration

Activate the relevant skills as soon as the task signal is clear — don't wait for the user to name a skill explicitly. Every skill named below exists in `skills/`; nothing here is aspirational.

| Task signal | Activate |
|---|---|
| Building or modifying UI | `frontend-ui-engineering` (+ `animation` if motion/transitions are involved) |
| Building or reviewing an e-commerce store | `ecommerce-storefront` + `payments-and-billing` + `database-design` + `seo-and-discoverability` |
| Implementing AI/agent/chat/RAG/voice features | `ai-integration-and-agents` + `security-and-hardening` (only when AI solves a real requirement — not by default) |
| Exposing an API/interface | `api-and-interface-design` |
| Consuming a third-party API or webhook | `api-integration` |
| Modifying authentication, authorization, or handling sensitive data | `security-and-hardening` |
| Optimizing performance | `performance-optimization` + `browser-testing-with-devtools` for real measurement |
| Improving SEO or discoverability | `seo-and-discoverability` |
| Fixing bugs | `reviewing-code` + `writing-tests` + `browser-testing-with-devtools` |
| Preparing a production deployment | `security-and-hardening` + `performance-optimization` + `writing-tests` + `ci-cd-and-automation` + `shipping-and-launch` |
| Designing database schema, indexes, or migrations | `database-design` |
| Instrumenting logging, metrics, or alerting | `observability-and-instrumentation` |
| Deciding where a file/component/hook belongs | `organizing-project-files` |
| Using a cloud service's CLI vs. its dashboard | `using-cli-tools` |
| Writing a commit message or PR description | `writing-commits` |

Use only the skills that materially improve the current task — this table is a floor for what to activate automatically, not a mandate to load all of them on every task.

## Project Awareness

Before modifying an existing project, inspect (don't assume): `package.json` and framework, overall architecture, `src`/`app` structure, components, APIs, database schema, authentication, integrations, environment variables, deployment configuration, and the existing design system. Identify what already works and protect it. Never perform a large rewrite without a stated technical reason — matching the Existing-Project Protection constraint below.

## Execution Loops

**Substantial task** (bug fix, small feature, focused change):
`DISCOVER → PLAN → IMPLEMENT → VERIFY → REVIEW → IMPROVE`

**Major feature** (new capability, new page type, new integration):
`DISCOVER → ARCHITECT → DESIGN → IMPLEMENT → TEST → SECURITY REVIEW → PERFORMANCE REVIEW → FINAL QA`

Pick the loop that matches the size of the change; don't run the major-feature loop for a one-line fix, and don't skip straight to IMPLEMENT for a major feature.

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

## Evidence-Based Completion

Before marking any quality-gate category (Functionality, Design, UX, Responsiveness, Accessibility, Security, Performance, SEO, Code Quality, Maintainability) as satisfied, have actual evidence for it — never "everything is perfect" with nothing behind it. Evidence looks like: tests executed (and their result), build completed, lint completed, routes verified, API behavior verified, responsive behavior checked at real breakpoints, security issues checked, performance issues checked.

**If something was not actually checked, say `NOT VERIFIED` for that item, explicitly.** A category with no evidence is not a passed category — it's an unknown, and it must be reported as one.

## Change Discipline

Before a destructive or hard-to-reverse change (deleting a working feature, overwriting a component someone depends on, dropping a database column, replacing a working integration), state the impact and choose the least destructive option that still satisfies the requirement.

Never, regardless of how it would speed up the task:
- Delete working features without a stated reason
- Overwrite credentials or environment variables
- Expose secrets (to the frontend, to logs, to version control)
- Fabricate data, responses, or test results
- Install a dependency the task doesn't actually need
- Introduce architecture (a new service, a new state layer, a new abstraction) the requirement doesn't call for

## Non-Negotiable Constraints

- **No fake functionality.** Never fabricate analytics, orders, customers, payments, API responses, AI responses, or database records to make something look finished. If real functionality isn't wired up yet, label demo data as demo data, visibly, not as if it were production.
- **No overengineering.** No unnecessary dependencies, microservices, abstractions, state-management layers, animations, AI, or infrastructure. Use the simplest architecture that reliably satisfies the actual requirement — this is the same principle `performance-optimization`'s "don't optimize before you have evidence" and `api-and-interface-design`'s addition-over-modification guidance both apply in their own domains.
- **Existing-project protection.** When modifying a project that already has working code: inspect it first, identify what already works, and change only what's necessary. Preserve existing integrations, environment variables, database structure, auth, and deployment config unless there's a clear, stated reason to change them. Don't rewrite something just to make the code look different.

## Final Audit Report Format

Before declaring a substantial task complete, report against this structure — every section grounded in what was actually done, not asserted:

```
COMPLETED      — what was actually implemented
SKILLS USED    — which skills were activated and why
VERIFICATION   — what was tested or verified (mark anything untested NOT VERIFIED)
SECURITY       — what security checks were performed
PERFORMANCE    — what performance checks were performed
ACCESSIBILITY  — what accessibility checks were performed
SEO            — what SEO checks were performed (where relevant)
REMAINING      — anything that still needs attention, stated plainly
```

## Master Rule

Optimize for *"how can I deliver the highest-quality production system with the smallest necessary complexity"* — not for *"how quickly can I generate code."* Think before coding. Inspect before modifying. Test before declaring success. Never claim something is complete unless it has actually been implemented and verified.

## This Is a Standard, Not a Reference

This file isn't background reading — it's the operating standard for AURIX work in any Claude Code session, on any AURIX project repository, regardless of which one happens to be open. Follow it without being reminded: read it at session start for a substantial task, activate skills automatically per the table above, run the execution loop that matches the change's size, and report completion with evidence per the format above. If a step here doesn't fit the task at hand, say so and explain why — don't silently skip it.
