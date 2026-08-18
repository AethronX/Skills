# Agent Skills Repository

A collection of skills for AI coding agents focused on modern web development workflows.

## Repository Structure

Skills are packaged instructions and reference files that extend agent capabilities.

```
skills/
  {skill-name}/              # kebab-case directory name
    SKILL.md                 # Required: skill definition
    {REFERENCE}.md           # Optional: additional context loaded on-demand
```

## Skill Format

### Directory Naming
- Skill directory: kebab-case (e.g., `using-cli-tools`, `writing-tests`)
- SKILL.md: Always uppercase, always this exact filename
- Reference files: UPPERCASE.md (e.g., `PATTERNS.md`, `MOCKING.md`)

### SKILL.md Structure

```markdown
---
name: {skill-name}
description: {One sentence describing what the skill does and when to use it. Include trigger phrases.}
---

# {Skill Title}

{Instructions for the agent to follow when this skill is activated.}
```

## Available Skills

| Skill | Description |
|-------|-------------|
| `using-cli-tools` | CLI preferences for GitHub, Supabase, Vercel, Netlify, Cloudflare, Prisma, Stripe, Docker |
| `organizing-project-files` | Project structure conventions for React/Next.js applications |
| `writing-commits` | Conventional Commits format, PR templates, branch naming |
| `reviewing-code` | Structured code review with prioritized feedback (read-only) |
| `security-and-hardening` | OWASP-based security hardening, threat modeling, secrets/dependency hygiene |
| `performance-optimization` | Measure-first performance workflow, Core Web Vitals, N+1 fixes |
| `frontend-ui-engineering` | Production-quality accessible UI patterns, WCAG 2.1 AA |
| `ci-cd-and-automation` | Quality-gate CI pipelines, feature flags, staged rollouts |
| `api-and-interface-design` | Contract-first REST/API design, idempotency-key handling |
| `browser-testing-with-devtools` | Live browser verification via Chrome DevTools MCP |
| `observability-and-instrumentation` | Structured logging, RED/USE metrics, OpenTelemetry tracing |
| `shipping-and-launch` | Pre-launch checklists, staged rollout thresholds, rollback plans |
| `animation` | Motion/animation design: performance, accessibility, CSS vs. View Transitions vs. Framer Motion |
| `seo-and-discoverability` | Technical SEO, structured data, and AEO for AI answer engines |
| `payments-and-billing` | Payment/subscription correctness, webhooks, idempotency, PCI-scope avoidance |
| `database-design` | Schema design, indexing, migration safety, multi-tenancy |
| `ai-integration-and-agents` | Building chat/agent/RAG/voice AI features (pairs with security-and-hardening) |
| `ecommerce-storefront` | Product catalog, inventory correctness, cart, checkout, storefront search |
| `api-integration` | Consuming third-party APIs/webhooks: retries, rate limits, circuit breakers |
| `writing-tests` | Testing strategy for unit, integration, and e2e tests |

Skills are original unless noted in their `SKILL.md` frontmatter (`source:` field) and [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md).

## Best Practices for Context Efficiency

Skills are loaded on-demand — only the skill name and description are loaded at startup. The full `SKILL.md` loads into context only when the agent decides the skill is relevant.

**Keep SKILL.md lean:**
- Core instructions only
- Reference detailed content in separate files
- Let the agent load additional files as needed

**Use reference files for:**
- Detailed examples
- Framework-specific patterns
- Extended documentation
