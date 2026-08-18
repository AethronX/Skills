# Advanced Project Patterns

## Next.js App Router

```
app/
├── (marketing)/            # Route group (no URL impact)
│   ├── page.tsx            # /
│   ├── about/page.tsx      # /about
│   └── layout.tsx          # Shared marketing layout
├── (app)/                  # Route group for authenticated app
│   ├── dashboard/page.tsx  # /dashboard
│   ├── settings/page.tsx   # /settings
│   └── layout.tsx          # Shared app layout (with sidebar)
├── api/
│   └── webhooks/
│       └── stripe/route.ts # /api/webhooks/stripe
├── _components/            # Private folder (not routed)
│   └── Header.tsx
├── layout.tsx              # Root layout
└── globals.css
```

### App Router Conventions

| File | Purpose |
|------|---------|
| `page.tsx` | Route UI |
| `layout.tsx` | Shared UI wrapper |
| `loading.tsx` | Loading state |
| `error.tsx` | Error boundary |
| `not-found.tsx` | 404 UI |
| `route.ts` | API endpoint |

### Server vs Client Components

```
components/
├── server/                 # Server Components (default)
│   ├── UserProfile.tsx     # Can fetch data directly
│   └── ProductList.tsx
└── client/                 # Client Components ('use client')
    ├── AddToCart.tsx       # Needs interactivity
    └── SearchInput.tsx     # Needs browser APIs
```

Or use naming convention:
- `Component.tsx` → Server (default)
- `Component.client.tsx` → Client

## Barrel Files (Index Exports)

Use sparingly—can hurt tree-shaking and cause circular dependencies.

**Good use**: Public API for a feature module

```typescript
// components/ui/index.ts
export { Button } from './Button'
export { Input } from './Input'
export { Modal } from './Modal'

// Usage
import { Button, Input } from '@/components/ui'
```

**Avoid**: Deep barrels that re-export everything

```typescript
// ❌ Don't do this at the root
// components/index.ts
export * from './ui'
export * from './forms'
export * from './layouts'
```

## Monorepo Structure

### Turborepo / pnpm Workspaces

```
my-monorepo/
├── apps/
│   ├── web/                # Next.js app
│   │   ├── app/
│   │   └── package.json
│   ├── api/                # Backend service
│   │   ├── src/
│   │   └── package.json
│   └── mobile/             # React Native (optional)
├── packages/
│   ├── ui/                 # Shared component library
│   │   ├── src/
│   │   └── package.json
│   ├── database/           # Prisma schema + client
│   │   ├── prisma/
│   │   └── package.json
│   ├── config-eslint/      # Shared ESLint config
│   ├── config-typescript/  # Shared tsconfig
│   └── utils/              # Shared utilities
├── package.json
├── pnpm-workspace.yaml
└── turbo.json
```

### Package Naming

```json
// packages/ui/package.json
{
  "name": "@acme/ui",
  "main": "./src/index.ts",
  "types": "./src/index.ts"
}

// apps/web/package.json
{
  "dependencies": {
    "@acme/ui": "workspace:*",
    "@acme/database": "workspace:*"
  }
}
```

## Feature-Based Structure (Vertical Slices)

For larger apps, organize by feature instead of type:

```
src/
├── features/
│   ├── auth/
│   │   ├── components/
│   │   │   ├── LoginForm.tsx
│   │   │   └── SignupForm.tsx
│   │   ├── hooks/
│   │   │   └── useAuth.ts
│   │   ├── api/
│   │   │   └── auth.ts
│   │   ├── types.ts
│   │   └── index.ts         # Public API
│   ├── dashboard/
│   │   ├── components/
│   │   ├── hooks/
│   │   └── index.ts
│   └── billing/
│       ├── components/
│       ├── hooks/
│       └── index.ts
├── shared/                   # Truly shared code
│   ├── components/
│   ├── hooks/
│   └── utils/
└── app/                      # Next.js routes
```

**Rule**: Features can import from `shared/`, but not from each other.
