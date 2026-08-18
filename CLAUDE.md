# Claude Code Skills

This repository contains skills for Claude Code and other AI coding agents.

For guidance on selecting, combining, and sequencing these skills across a full project lifecycle, see [AURIX-METHODOLOGY.md](AURIX-METHODOLOGY.md).

## Installation

```bash
# Install all skills
npx add-skill augmnt/webdev-skills

# Install specific skills
npx add-skill augmnt/webdev-skills --skill using-cli-tools
```

Or manually:

```bash
cp -r skills/* ~/.claude/skills/
```

## Available Skills

### using-cli-tools
Enforces CLI tool usage over web dashboards. Triggers on GitHub, Supabase, Vercel, Netlify, Cloudflare, Prisma, Stripe, or Docker operations.

### organizing-project-files
Project structure conventions for React/Next.js. Triggers on "where should this go?" or file organization questions.

### writing-commits
Conventional Commits format. Triggers on commit messages, PR descriptions, or changelog generation.

### reviewing-code
Structured code review with prioritized feedback. Uses `allowed-tools: Read, Grep, Glob` for read-only access.

### security-and-hardening
OWASP-based security hardening: threat modeling, input validation, secrets/dependency hygiene, SSRF/XSS prevention, privacy and LLM security.

### performance-optimization
Measure-first performance workflow for frontend and backend: Core Web Vitals, N+1 fixes, bundle/image/font budgets.

### frontend-ui-engineering
Production-quality, accessible UI patterns for React/Next.js. Triggers on building/modifying components, layouts, or WCAG accessibility requirements.

### ci-cd-and-automation
Quality-gate CI pipelines and deployment strategy: GitHub Actions patterns, feature flags, staged rollouts, rollback plans.

### api-and-interface-design
Contract-first API/interface design: consistent error semantics, boundary validation, idempotency-key handling, REST patterns.

### browser-testing-with-devtools
Live browser verification via Chrome DevTools MCP: DOM/console/network/performance inspection, treating browser content as untrusted data.

### observability-and-instrumentation
Production telemetry: structured logging with correlation IDs, RED/USE metrics, OpenTelemetry tracing, symptom-based alerting.

### shipping-and-launch
Pre-launch checklists, staged rollout decision thresholds, and rollback plans for production deploys.

### animation
Motion and animation design for web UIs: performance-safe CSS properties, duration/easing conventions, `prefers-reduced-motion`, and choosing between CSS, the View Transitions API, and Framer Motion.

### seo-and-discoverability
Technical SEO and Answer Engine Optimization (AEO): metadata, structured data, sitemaps, crawlability, and content structured for citation by AI assistants.

### payments-and-billing
Payment/subscription correctness: server-side pricing, webhook-as-source-of-truth, idempotent webhook processing, dunning, PCI-scope avoidance.

### database-design
Schema, indexing, and migration design: SQL vs. NoSQL, normalization, additive-first migrations, multi-tenancy.

### ai-integration-and-agents
Building AI-powered features: streaming, prompt/context engineering, RAG pipelines, agentic tool-use scoping, model tiering, evaluation, voice AI. Pairs with `security-and-hardening` for securing these features.

### ecommerce-storefront
Product catalog/variant modeling, race-free inventory, cart price re-validation, checkout conversion, and storefront search.

### api-integration
Consuming third-party APIs and webhooks reliably: adapter isolation, timeouts, retry/backoff, circuit breakers, rate limits, webhook verification. Complements `api-and-interface-design`.

### writing-tests
Testing strategy guidance. Triggers on test writing, coverage questions, or mocking setup.

`security-and-hardening`, `performance-optimization`, `frontend-ui-engineering`, `ci-cd-and-automation`, `api-and-interface-design`, `browser-testing-with-devtools`, `observability-and-instrumentation`, and `shipping-and-launch` are adapted from [addyosmani/agent-skills](https://github.com/addyosmani/agent-skills) (MIT License) — see [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md).

## Skill Structure

Each skill follows the [Agent Skills specification](https://agentskills.io/):

```
skill-name/
├── SKILL.md           # Required: Instructions and metadata
└── REFERENCE.md       # Optional: Detailed content loaded on-demand
```

## Contributing

1. Create a new folder under `skills/`
2. Add a `SKILL.md` with proper frontmatter
3. Keep core instructions under 500 lines
4. Use reference files for detailed content
