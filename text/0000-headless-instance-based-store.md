- Start Date: 2025-12-30
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Introduce `createStore` API for creating headless React instances that accept hooks and expose state for external consumption—enabling React's full hook system for state management outside components.

# Basic example

```jsx
const counterStore = createStore(() => {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    console.log('Count changed:', count);
  }, [count]);
  
  const increment = () => setCount(c => c + 1);

  return {
    count,
    increment,
  };
});

// Outside React
counterStore.subscribe(state => console.log(state.count));
counterStore.current.increment(); // Count changed: 1

// Inside React
function Counter() {
  const { count, increment } = useStore(counterStore);
  return <button onClick={increment}>{count}</button>;
}
```

# Motivation

1. **Unified mental model**: Same hooks API for component state and global state.

2. **Full React capabilities**: Unlike simplified state libraries, headless stores retain effects, memoization, Suspense, and refs.

# Detailed design

## `createStore<T>(hook: () => T): Store<T>`

```typescript
interface Store<T> {
  readonly current: T;
  subscribe(listener: (state: T) => void): () => void;
  destroy(): void;
}
```

## Hook behavior

Most hooks work as in components, with some exceptions: `useContext` always returns the default value (no Provider hierarchy exists in a headless instance), `use(context)` performs the same way, `use(promise)` should directly throw an error.

## Component integration

`useStore` consumes store state using `useSyncExternalStore` like tech internally.

# Drawbacks

- **Overlap**: `useSyncExternalStore` already handles external state; this adds another pattern
- **Complexity**: Requires maintaining a headless reconciler that stays in sync with React internals
- **Misuse potential**: Memory leaks if `destroy()` not called; `useContext` behavior may surprise users
- **Ecosystem impact**: May fragment state management library ecosystem

# Alternatives

just do nothing.

# Adoption strategy

Non-breaking addition as it do not make changes to existing APIs. Can create a polyfill based on [react-nil](https://github.com/pmndrs/react-nil) or similar libraries.  

# How we teach this

"React hooks, but for global state".

# Unresolved questions

- How should `useTransition` / `useDeferredValue` behave without UI?
- May provide some degree of `use(promise)` support.
- DevTools integration for headless store state?
