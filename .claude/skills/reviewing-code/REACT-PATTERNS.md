# React Patterns to Review

## Hooks Issues

### useEffect Dependencies

```typescript
// 🚨 Missing dependency
useEffect(() => {
  fetchUser(userId)
}, []) // userId missing

// ✅ Correct
useEffect(() => {
  fetchUser(userId)
}, [userId])

// ✅ Or use a ref if you intentionally want to skip
const userIdRef = useRef(userId)
useEffect(() => {
  fetchUser(userIdRef.current)
}, [])
```

### Stale Closures

```typescript
// 🚨 Stale closure in event handler
const [count, setCount] = useState(0)
useEffect(() => {
  const id = setInterval(() => {
    setCount(count + 1) // Always uses initial count
  }, 1000)
  return () => clearInterval(id)
}, [])

// ✅ Use functional update
setCount(prev => prev + 1)
```

### Unnecessary useEffect

```typescript
// 🚨 Derived state doesn't need useEffect
const [items, setItems] = useState([])
const [total, setTotal] = useState(0)
useEffect(() => {
  setTotal(items.reduce((sum, i) => sum + i.price, 0))
}, [items])

// ✅ Just compute it
const total = items.reduce((sum, i) => sum + i.price, 0)

// ✅ Or useMemo if expensive
const total = useMemo(
  () => items.reduce((sum, i) => sum + i.price, 0),
  [items]
)
```

### Missing Cleanup

```typescript
// 🚨 Memory leak - no cleanup
useEffect(() => {
  const controller = new AbortController()
  fetch('/api/data', { signal: controller.signal })
    .then(res => res.json())
    .then(setData)
}, [])

// ✅ With cleanup
useEffect(() => {
  const controller = new AbortController()
  fetch('/api/data', { signal: controller.signal })
    .then(res => res.json())
    .then(setData)
    .catch(e => {
      if (e.name !== 'AbortError') throw e
    })
  return () => controller.abort()
}, [])
```

## useMemo / useCallback Misuse

```typescript
// 🚨 Premature optimization
const value = useMemo(() => items.length, [items])

// ✅ Just compute it
const value = items.length

// ✅ useMemo for expensive computations
const sorted = useMemo(
  () => items.sort((a, b) => complexCompare(a, b)),
  [items]
)

// ✅ useCallback for stable function identity (when passed to memoized children)
const handleClick = useCallback(() => {
  doSomething(id)
}, [id])
```

## State Management

### Prop Drilling

```typescript
// 🚨 Drilling through many levels
<App user={user}>
  <Layout user={user}>
    <Sidebar user={user}>
      <UserMenu user={user} />

// ✅ Use context for widely-shared state
const UserContext = createContext(null)
<UserContext.Provider value={user}>
  <App />
```

### State Location

```typescript
// 🚨 Lifting state too high
// App.tsx manages form state for deeply nested form

// ✅ Keep state close to where it's used
// Form.tsx manages its own state, only lifts on submit
```

## Component Patterns

### Key Prop Issues

```typescript
// 🚨 Index as key with dynamic list
{items.map((item, index) => (
  <Item key={index} {...item} />
))}

// ✅ Stable unique key
{items.map(item => (
  <Item key={item.id} {...item} />
))}
```

### Conditional Rendering

```typescript
// 🚨 Can render "0" if count is 0
{count && <Badge count={count} />}

// ✅ Explicit boolean
{count > 0 && <Badge count={count} />}

// ✅ Or ternary
{count ? <Badge count={count} /> : null}
```

## Accessibility Checklist

### Interactive Elements

- [ ] Buttons use `<button>`, not `<div onClick>`
- [ ] Links use `<a href>`, not `<span onClick>`
- [ ] Form inputs have associated `<label>`
- [ ] Custom controls have `role` and `aria-*` attributes

### Focus Management

- [ ] Focus visible on interactive elements (`:focus-visible`)
- [ ] Focus trapped in modals
- [ ] Focus returned after modal closes
- [ ] Skip links for keyboard navigation

### Semantic HTML

- [ ] Headings in logical order (h1 → h2 → h3)
- [ ] Lists use `<ul>`, `<ol>`, `<li>`
- [ ] Tables have `<th>` with `scope`
- [ ] Landmarks: `<main>`, `<nav>`, `<aside>`

### Images & Icons

- [ ] Images have `alt` text (or `alt=""` if decorative)
- [ ] Icon buttons have `aria-label`
- [ ] SVGs have `role="img"` and `aria-label` or are `aria-hidden`

### Common Issues

```typescript
// 🚨 Div as button
<div onClick={handleClick}>Submit</div>

// ✅ Semantic button
<button onClick={handleClick}>Submit</button>

// 🚨 Missing label
<input type="email" placeholder="Email" />

// ✅ With label
<label>
  Email
  <input type="email" />
</label>

// 🚨 Icon button without label
<button onClick={onClose}>
  <XIcon />
</button>

// ✅ With aria-label
<button onClick={onClose} aria-label="Close dialog">
  <XIcon aria-hidden />
</button>
```

## Performance Red Flags

- [ ] Large lists without virtualization
- [ ] Images without `loading="lazy"`
- [ ] Bundle includes unused dependencies
- [ ] Re-renders on every keystroke (check with React DevTools)
- [ ] Network requests in render (should be in useEffect or data fetching lib)
