# Web Developer Skills

A collection of skills for AI coding agents focused on modern web development workflows.

Skills follow the [Agent Skills](https://agentskills.io/) specification and work with [Claude Code](https://claude.ai/code), [OpenCode](https://opencode.ai), [Codex](https://developers.openai.com/codex), and [Cursor](https://cursor.com).

## Available Skills

### using-cli-tools

Enforces CLI tool usage over web dashboards for reproducibility and scriptability.

**Use when:** Working with GitHub, Supabase, Vercel, Netlify, Cloudflare, Prisma, Stripe, Docker, or any cloud service with a CLI.

**Includes:**
- Core tools: `gh`, `supabase`, `vercel`, `netlify`, `wrangler`, `prisma`, `stripe`, `docker compose`
- Additional: AWS CLI, Railway, Fly.io, ngrok, Turborepo, Planetscale, Neon, Drizzle, Bun

---

### organizing-project-files

Project structure conventions for React and Next.js applications.

**Use when:** Creating new files, deciding where components/hooks/utils should go, or asking "where does this belong?"

**Includes:**
- Standard folder layout
- Naming conventions  
- App Router patterns
- Monorepo structure
- Feature-based organization

---

### writing-commits

Consistent git history with Conventional Commits.

**Use when:** Writing commit messages, PR titles, PR descriptions, or branch names.

**Format:**
```
feat(auth): add Google OAuth login
fix(api): handle null user in profile endpoint
chore(deps): upgrade Next.js to 15
```

---

### reviewing-code

Structured code review with prioritized, actionable feedback.

**Use when:** Reviewing PRs, checking code quality, or auditing changes.

**Features:**
- Read-only mode (`allowed-tools: Read, Grep, Glob`)
- Prioritized feedback: 🚨 Blocker → ⚠️ Suggestion → 💭 Nit
- React/hooks patterns checklist
- Accessibility audit

---

### security-and-hardening

OWASP-based security hardening for web applications.

**Use when:** Handling user input, authentication, data storage, payments, or external/LLM integrations.

**Includes:**
- Threat modeling (STRIDE) and trust-boundary mapping
- OWASP Top 10 prevention patterns (injection, XSS, SSRF, broken auth/access control)
- Secrets management, dependency/supply-chain hygiene
- Data privacy (GDPR/CCPA) and LLM/AI security (OWASP LLM Top 10)

---

### performance-optimization

Measure-first performance workflow for frontend and backend.

**Use when:** Performance requirements exist, Core Web Vitals are below target, or you suspect a regression.

**Includes:**
- Core Web Vitals targets (LCP, INP, CLS) and a measure → identify → fix → verify → guard workflow
- N+1 query, bundle-size, and image/font optimization patterns
- Performance budgets and a strict keep-or-revert verification step

---

### frontend-ui-engineering

Production-quality, accessible UI patterns for React/Next.js.

**Use when:** Building or modifying user-facing interfaces and components.

**Includes:**
- Component architecture and state-management decision guide
- Avoiding the generic "AI aesthetic" (design-system adherence)
- WCAG 2.1 AA accessibility checklist and testing tools

---

### writing-tests

Practical testing strategy for web applications.

**Use when:** Writing tests, deciding what to test, setting up test infrastructure, or discussing coverage.

**Includes:**
- Testing pyramid guidance
- MSW setup for API mocking
- Factory functions and fixtures
- Coverage targets and exclusions

## Installation

### Quick Install

```bash
# Install all skills
npx add-skill augmnt/webdev-skills

# List available skills
npx add-skill augmnt/webdev-skills --list

# Install specific skills
npx add-skill augmnt/webdev-skills --skill using-cli-tools --skill writing-tests
```

### Claude Code

```bash
# Global (all projects)
cp -r skills/* ~/.claude/skills/

# Project-level (commit to share with team)
cp -r skills/* .claude/skills/
```

### OpenCode

```bash
cp -r skills/* ~/.config/opencode/skill/
```

### Codex

```bash
cp -r skills/* ~/.codex/skills/
```

### Cursor

```bash
cp -r skills/* ~/.cursor/skills/
```

Skills are automatically available once installed.

## Customization

These skills are designed to be opinionated starting points. Fork this repo and customize for your team:

- Add your preferred tools to `using-cli-tools`
- Adjust folder structure in `organizing-project-files` 
- Modify commit conventions in `writing-commits`
- Add team-specific review criteria to `reviewing-code`
- Update testing tools in `writing-tests`

## Contributing

Contributions welcome! Please:

1. Follow the [Agent Skills spec](https://agentskills.io/)
2. Keep SKILL.md files under 500 lines
3. Use reference files for detailed content
4. Test with multiple agents before submitting

## License

MIT

Skills in this repository are original unless noted. `security-and-hardening`, `performance-optimization`, and `frontend-ui-engineering` are adapted from [addyosmani/agent-skills](https://github.com/addyosmani/agent-skills) (MIT) — see [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md) for full attribution.
