---
name: animation
description: Guides motion and animation design for web UIs. Use when adding transitions, page/route animations, list reordering, loading/skeleton motion, or micro-interactions. Use when choosing between CSS transitions, the View Transitions API, and a JS animation library, or when reviewing animation code for jank or accessibility issues.
---

# Animation

## Overview

Motion communicates meaning — it should show *what changed*, not decorate the page. An animation that doesn't answer "what happened, and where did it go?" is noise, and noise costs performance and accessibility for nothing in return. Every animation must be justifiable in one sentence: "this shows the item was deleted," "this shows the panel is loading," "this shows navigation moved one level deeper."

## When to Use

- Adding enter/exit transitions for elements, modals, or toasts
- Animating route or view changes
- Reordering or filtering a list and wanting continuity between old and new positions
- Building loading states (skeletons, progress, spinners)
- Adding micro-interactions (button press, hover, focus feedback)
- Reviewing existing animation code for performance or accessibility problems

**When NOT to use:** Purely decorative motion with no state change to communicate (autoplaying background animations, gratuitous parallax). If removing the animation loses no information, it's decoration, not communication — and it's the first thing to cut under a performance or accessibility budget.

## Core Principle: Animate Only Cheap Properties

The browser can animate `transform` and `opacity` on the compositor thread, off the main thread, without triggering layout or paint. Everything else is expensive:

```css
/* GOOD: compositor-only, no layout/paint */
.enter {
  transform: translateY(8px) scale(0.98);
  opacity: 0;
  transition: transform 200ms ease-out, opacity 200ms ease-out;
}
.enter.visible {
  transform: translateY(0) scale(1);
  opacity: 1;
}

/* BAD: animating layout-triggering properties causes jank */
.bad-enter {
  height: 0;
  margin-top: -20px;
  transition: height 200ms, margin-top 200ms; /* forces layout every frame */
}
```

If you must animate a layout change (e.g., an accordion's height), either measure the target height with JS and animate to a fixed pixel value, or use `height: auto` with the [`calc-size()`](https://developer.mozilla.org/en-US/docs/Web/CSS/calc-size) / grid-template-rows trick, or accept the cost only for infrequent, non-interactive transitions.

## Duration and Easing Conventions

| Interaction type | Duration | Easing |
|---|---|---|
| Micro-interaction (hover, press, toggle) | 100–150ms | `ease-out` |
| Element enter/exit (toast, tooltip, dropdown) | 150–250ms | enter: `ease-out`, exit: `ease-in` |
| Panel/modal/page transition | 250–400ms | `ease-in-out` or a custom cubic-bezier |
| Large layout reflow (list reorder) | 300–500ms | `ease-in-out` |

**Enter is slower than it looks, exit is faster than it looks:** entering elements should decelerate into place (`ease-out`); exiting elements should accelerate away (`ease-in`) since the user's attention has already moved on. Using the same easing for both is the most common "off" feeling in custom animations.

Don't invent arbitrary durations. Pick from a scale (100/150/200/300/400/500ms) the same way spacing or type scales are fixed — see the `frontend-ui-engineering` skill's guidance on not inventing ad hoc values.

## Accessibility: `prefers-reduced-motion`

Some users get vestibular symptoms (dizziness, nausea) from motion, especially large-scale parallax, zoom, and spin effects. Respect their OS-level preference — this is not optional polish:

```css
@media (prefers-reduced-motion: reduce) {
  * {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

For JS-driven animation, check the same media query and skip or shorten the animation rather than disabling functionality:

```typescript
const prefersReducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches;

const duration = prefersReducedMotion ? 0 : 250;
```

**Other accessibility rules:**
- Never convey required information through motion alone (a shake alone isn't an error state — pair it with text/icon/color, per `frontend-ui-engineering`).
- Nothing should flash more than 3 times per second (seizure risk).
- Auto-playing motion that loops indefinitely (carousels, background video) needs a visible pause control.
- Focus must land somewhere sensible after an animated transition — don't let it get lost on a node that unmounted mid-animation.

## Choosing the Right Tool

```
What are you animating?
├── Simple state toggle (hover, open/closed, active) on one element
│   → CSS transition or CSS animation. No JS needed.
├── Enter/exit where the element mounts/unmounts from the DOM
│   → JS animation library (Framer Motion `AnimatePresence`) — CSS alone can't
│     animate an element that's already been removed from the DOM.
├── Shared continuity between two different views/routes (an image that appears
│   to "grow" from a thumbnail into a detail page)
│   → View Transitions API (native) if browser support matches your audience;
│     otherwise a library-based shared-layout animation.
├── List reordering / filtering where items should visibly move to new positions
│   → Framer Motion `layout` prop, or the FLIP technique manually.
└── Complex choreography (staggered entrances, gesture-driven, physics)
    → JS animation library (Framer Motion, GSAP). Don't hand-roll spring physics.
```

## CSS-Only Patterns

```css
/* Fade + slide in on mount (requires the class to be added a frame after mount) */
.card {
  opacity: 0;
  transform: translateY(12px);
  transition: opacity 200ms ease-out, transform 200ms ease-out;
}
.card.is-visible {
  opacity: 1;
  transform: translateY(0);
}

/* Skeleton loading shimmer */
.skeleton {
  background: linear-gradient(90deg, #eee 25%, #f5f5f5 37%, #eee 63%);
  background-size: 400% 100%;
  animation: shimmer 1.4s ease infinite;
}
@keyframes shimmer {
  0% { background-position: 100% 50%; }
  100% { background-position: 0% 50%; }
}

/* Respect reduced motion even for keyframe animations */
@media (prefers-reduced-motion: reduce) {
  .skeleton { animation: none; background-position: 0% 50%; }
}
```

## The View Transitions API (Native, Cross-DOM)

`document.startViewTransition()` lets the browser cross-fade/morph between two DOM states — including across a full page navigation — without a JS animation library. Supported in Chromium and Firefox; Safari support is landing. Always feature-detect and fall back gracefully:

```typescript
function updateWithTransition(updateDOM: () => void) {
  if (!document.startViewTransition) {
    updateDOM(); // no transition support — just apply the change
    return;
  }
  document.startViewTransition(() => updateDOM());
}
```

```css
/* Name the element you want to persist across the transition */
.product-thumbnail {
  view-transition-name: product-hero;
}

/* Customize the default cross-fade */
::view-transition-old(product-hero),
::view-transition-new(product-hero) {
  animation-duration: 300ms;
}
```

For Next.js App Router, wrap the navigation-triggering state update in `startTransition` (or a router transition) so the browser can associate the DOM mutation with the view transition; check your router/framework version for built-in support before hand-rolling this.

## Framer Motion (React) Patterns

```tsx
import { motion, AnimatePresence } from 'framer-motion';

// Enter/exit for a mounting/unmounting element — CSS alone cannot do this
function Toast({ message, onDismiss }: ToastProps) {
  return (
    <AnimatePresence>
      {message && (
        <motion.div
          initial={{ opacity: 0, y: 16 }}
          animate={{ opacity: 1, y: 0 }}
          exit={{ opacity: 0, y: 16 }}
          transition={{ duration: 0.2, ease: 'easeOut' }}
        >
          {message}
        </motion.div>
      )}
    </AnimatePresence>
  );
}

// Layout animation — items animate to their new position automatically
function TaskList({ tasks }: { tasks: Task[] }) {
  return (
    <ul>
      {tasks.map(task => (
        <motion.li key={task.id} layout transition={{ duration: 0.3 }}>
          {task.title}
        </motion.li>
      ))}
    </ul>
  );
}

// Staggered entrance for a list
const container = {
  hidden: { opacity: 0 },
  show: { opacity: 1, transition: { staggerChildren: 0.05 } },
};
const item = {
  hidden: { opacity: 0, y: 8 },
  show: { opacity: 1, y: 0 },
};

function StaggeredList({ items }: { items: string[] }) {
  return (
    <motion.ul variants={container} initial="hidden" animate="show">
      {items.map(i => <motion.li key={i} variants={item}>{i}</motion.li>)}
    </motion.ul>
  );
}

// Respecting reduced motion inside Framer Motion
import { useReducedMotion } from 'framer-motion';

function Panel({ children }: { children: React.ReactNode }) {
  const shouldReduceMotion = useReducedMotion();
  return (
    <motion.div
      initial={{ opacity: 0, scale: shouldReduceMotion ? 1 : 0.96 }}
      animate={{ opacity: 1, scale: 1 }}
      transition={{ duration: shouldReduceMotion ? 0 : 0.2 }}
    >
      {children}
    </motion.div>
  );
}
```

**Rule:** always use `AnimatePresence` (or an equivalent) for elements that unmount — a plain CSS `transition` cannot play once the node is gone from the DOM. This is the single most common animation bug: exit animation "not working" because the element was removed before the transition class was ever applied.

## Performance Checklist

- [ ] Only `transform` and `opacity` are animated on hot paths (scrolling, dragging, frequent re-renders)
- [ ] No animation running on a hidden/off-screen element (`content-visibility: auto` or pause when out of viewport)
- [ ] Long lists animate with `will-change: transform` only while actively animating, then remove it (permanent `will-change` wastes GPU memory)
- [ ] Animations don't block input — nothing animating should also be doing synchronous heavy work on the main thread (see `performance-optimization`)
- [ ] `prefers-reduced-motion` is honored everywhere motion appears, not just in one component

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "It looks more polished with motion" | Unjustified motion reads as noise, not polish, and costs battery/performance on every interaction. |
| "Users will just disable animations if they don't like them" | Most users never touch that OS setting; you must query `prefers-reduced-motion`, not assume they opted out. |
| "CSS transition isn't firing on exit" | The element was removed from the DOM before the transition class could apply — you need an unmount-aware tool (`AnimatePresence` or a delayed removal). |
| "We'll just animate height directly" | Animating `height`/`width`/`top`/`left` forces layout every frame. Animate `transform`/`opacity`, or measure and animate to a fixed value. |
| "Everyone loves parallax" | Large-scale motion effects are a common vestibular trigger. Gate them behind `prefers-reduced-motion` or avoid them for content-heavy pages. |

## Red Flags

- Animating `width`, `height`, `top`, `left`, or `margin` on interactive or frequent transitions
- No `prefers-reduced-motion` handling anywhere in the codebase
- Exit animations attempted with plain CSS on a conditionally-rendered element
- `will-change` applied permanently to many elements
- Motion used as the *only* signal for an error, success, or required state change
- Content that flashes more than 3 times per second
- Auto-playing looped motion (carousel, video) with no pause control

## Verification

After adding or reviewing animation:

- [ ] The animation is describable in one sentence as communicating a real state change
- [ ] Only `transform`/`opacity` animate on any path that runs during scroll, drag, or frequent re-render
- [ ] `prefers-reduced-motion: reduce` is honored (durations near-zero or motion skipped, not just "slightly shorter")
- [ ] Enter/exit for conditionally-rendered elements uses a tool that supports unmount animation, not bare CSS transitions
- [ ] No content flashes more than 3 times per second
- [ ] Tested with the OS "reduce motion" setting turned on
