# Mocking & Test Utilities

## MSW (Mock Service Worker)

The standard for API mocking in tests and development.

### Setup

```typescript
// mocks/handlers.ts
import { http, HttpResponse } from 'msw'

export const handlers = [
  http.get('/api/user', () => {
    return HttpResponse.json({
      id: '1',
      name: 'John Doe',
      email: 'john@example.com'
    })
  }),

  http.post('/api/login', async ({ request }) => {
    const { email, password } = await request.json()
    
    if (email === 'test@example.com') {
      return HttpResponse.json({ token: 'fake-token' })
    }
    
    return HttpResponse.json(
      { error: 'Invalid credentials' },
      { status: 401 }
    )
  }),

  http.get('/api/posts', ({ request }) => {
    const url = new URL(request.url)
    const limit = url.searchParams.get('limit') || '10'
    
    return HttpResponse.json({
      posts: Array.from({ length: Number(limit) }, (_, i) => ({
        id: i + 1,
        title: `Post ${i + 1}`
      }))
    })
  })
]
```

### Test Setup (Vitest)

```typescript
// mocks/server.ts
import { setupServer } from 'msw/node'
import { handlers } from './handlers'

export const server = setupServer(...handlers)

// vitest.setup.ts
import { beforeAll, afterEach, afterAll } from 'vitest'
import { server } from './mocks/server'

beforeAll(() => server.listen({ onUnhandledRequest: 'error' }))
afterEach(() => server.resetHandlers())
afterAll(() => server.close())
```

### Override Handlers in Tests

```typescript
import { http, HttpResponse } from 'msw'
import { server } from '../mocks/server'

test('handles server error', async () => {
  server.use(
    http.get('/api/user', () => {
      return HttpResponse.json(
        { error: 'Internal server error' },
        { status: 500 }
      )
    })
  )

  render(<UserProfile />)
  
  await waitFor(() => {
    expect(screen.getByText('Something went wrong')).toBeInTheDocument()
  })
})
```

## Test Fixtures

### Factory Functions

```typescript
// tests/factories/user.ts
interface User {
  id: string
  name: string
  email: string
  role: 'admin' | 'user'
  createdAt: Date
}

export function createUser(overrides: Partial<User> = {}): User {
  return {
    id: crypto.randomUUID(),
    name: 'Test User',
    email: 'test@example.com',
    role: 'user',
    createdAt: new Date(),
    ...overrides
  }
}

// Usage
const admin = createUser({ role: 'admin' })
const users = Array.from({ length: 5 }, () => createUser())
```

### Fixture Files

```typescript
// tests/fixtures/products.ts
export const products = [
  { id: '1', name: 'Widget', price: 9.99, stock: 100 },
  { id: '2', name: 'Gadget', price: 19.99, stock: 50 },
  { id: '3', name: 'Gizmo', price: 29.99, stock: 0 } // Out of stock
]

export const emptyCart = { items: [], total: 0 }

export const cartWithItems = {
  items: [
    { productId: '1', quantity: 2 },
    { productId: '2', quantity: 1 }
  ],
  total: 39.97
}
```

## Custom Render

Wrap components with providers for testing:

```typescript
// tests/utils.tsx
import { render, RenderOptions } from '@testing-library/react'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { AuthProvider } from '@/contexts/auth'

interface CustomRenderOptions extends RenderOptions {
  user?: User | null
  queryClient?: QueryClient
}

export function renderWithProviders(
  ui: React.ReactElement,
  {
    user = null,
    queryClient = new QueryClient({
      defaultOptions: {
        queries: { retry: false }
      }
    }),
    ...options
  }: CustomRenderOptions = {}
) {
  function Wrapper({ children }: { children: React.ReactNode }) {
    return (
      <QueryClientProvider client={queryClient}>
        <AuthProvider initialUser={user}>
          {children}
        </AuthProvider>
      </QueryClientProvider>
    )
  }

  return render(ui, { wrapper: Wrapper, ...options })
}

// Re-export everything
export * from '@testing-library/react'
export { renderWithProviders as render }
```

## Coverage Guidelines

### Targets

| Type | Target | Notes |
|------|--------|-------|
| Statements | 70-80% | Not a hard rule |
| Branches | 70-80% | Focus on important branches |
| Critical paths | 100% | Auth, payments, data mutations |

### What to Exclude

```typescript
// vitest.config.ts
export default defineConfig({
  test: {
    coverage: {
      exclude: [
        'node_modules/',
        'tests/',
        '**/*.d.ts',
        '**/*.config.*',
        '**/types/',
        'src/mocks/',
        // Generated code
        'src/generated/',
        // Entry points
        'src/main.tsx',
        'src/index.ts'
      ]
    }
  }
})
```

### Coverage Anti-patterns

```typescript
// ❌ Testing implementation to hit coverage
test('calls internal method', () => {
  const spy = vi.spyOn(service, '_internalMethod')
  service.publicMethod()
  expect(spy).toHaveBeenCalled()
})

// ✅ Test behavior instead
test('returns processed data', () => {
  const result = service.publicMethod()
  expect(result).toEqual({ processed: true })
})
```

## Mocking Modules

### Vitest

```typescript
// Mock entire module
vi.mock('@/lib/analytics', () => ({
  track: vi.fn(),
  identify: vi.fn()
}))

// Mock with actual implementation for some exports
vi.mock('@/lib/utils', async () => {
  const actual = await vi.importActual('@/lib/utils')
  return {
    ...actual,
    formatDate: vi.fn(() => '2024-01-01')
  }
})

// Mock environment variables
vi.stubEnv('API_URL', 'http://test-api.com')
```

### Mocking Hooks

```typescript
// Mock a custom hook
vi.mock('@/hooks/useAuth', () => ({
  useAuth: () => ({
    user: { id: '1', name: 'Test' },
    isAuthenticated: true,
    login: vi.fn(),
    logout: vi.fn()
  })
}))

// Or use a factory for different states
const mockUseAuth = vi.fn()
vi.mock('@/hooks/useAuth', () => ({
  useAuth: () => mockUseAuth()
}))

test('shows login when unauthenticated', () => {
  mockUseAuth.mockReturnValue({ user: null, isAuthenticated: false })
  // ...
})
```

## Testing Async Code

```typescript
// Wait for element
await waitFor(() => {
  expect(screen.getByText('Loaded')).toBeInTheDocument()
})

// Wait for element to disappear
await waitForElementToBeRemoved(() => screen.queryByText('Loading...'))

// Find (with built-in waitFor)
const button = await screen.findByRole('button', { name: 'Submit' })

// User events (prefer over fireEvent)
import userEvent from '@testing-library/user-event'

test('form submission', async () => {
  const user = userEvent.setup()
  
  await user.type(screen.getByLabelText('Email'), 'test@example.com')
  await user.click(screen.getByRole('button', { name: 'Submit' }))
  
  await waitFor(() => {
    expect(screen.getByText('Success')).toBeInTheDocument()
  })
})
```
