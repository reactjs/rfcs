- Start Date: 2026-04-20
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Make `useId()` stable across Suspense retries of the same element — including during initial render — so it can serve as the default cache key for **element-scoped caches**.

# Basic example

```js
function useElementCache(compute) {
  const id = useId()
  if (!cache.has(id)) cache.set(id, compute())
  return cache.get(id)
}

function Product({ productId }) {
  // No user-supplied cache key. Lifecycle matches this element.
  const data = useElementCache(() => fetchProduct(productId))
  return <pre>{JSON.stringify(data)}</pre>
}
```

Today this fails: the component suspends on `fetchProduct`, React retries, `useId` returns a new value, the cache misses, the fetch runs again.

# Motivation

React already separates local from shared state:

|   | Local to element | Shared |
| --- | --- | --- |
| Sync state | `useState` | Redux, Zustand, … |
| Async data | **element cache *(missing)*** | TanStack Query, SWR, … |

The right-hand column is well-served. The left-hand column is blocked on a stable element identity, which is exactly what `useId` would be if it didn't reset on Suspense retries.

The payoff is zero-config component-level data fetching: users don't create cache keys, caches auto-evict on unmount, and the lifecycle matches the one they already know from `useState`. Without it, libraries ([Telefunc](https://telefunc.com) included) either push users onto shared-cache libraries for every fetch, or ask them to create cache keys by hand.

# Prior conversation

- https://github.com/facebook/react/issues/24669
- https://github.com/facebook/react/issues/24669#issuecomment-3761064054
- https://github.com/facebook/react/issues/24669#issuecomment-3763672850
