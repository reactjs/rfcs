# Granular Subscription State Architecture

Start Date: 2025-11-23  
RFC PR: (TBD)  
React Issue: (TBD)

## Summary

This RFC proposes adding granular subscription tracking to React state, enabling components to re-render only when the specific slice of state they use has changed. It introduces selector-based dependency tracking so that updates propagate only to components that read affected paths, reducing unnecessary re-renders and improving performance especially in large applications.

## Basic Example

```js
const store = createStore({
  user: { name: "John", age: 22 },
  theme: "dark",
});

const name = useGranular(store, state => state.user.name);

function Profile() {
  return <h1>{name}</h1>;
}

// Updating age should not re-render Profile
store.set(s => { s.user.age = 23 });
```

Because `<Profile />` is subscribed only to `user.name`, changes to `user.age` do not trigger a re-render.

## Motivation

React re-renders components when state changes propagate through the component tree. Even with modern state management libraries, unrelated state changes often trigger unnecessary rendering work. This causes performance bottlenecks in complex UI workloads such as dashboards, real-time visualizations, or large forms.

| Problem | Impact |
|---------|--------|
| Whole subtree updates | Excess CPU work |
| Unrelated components update | Jank & frames dropped |
| Requires manual memoization | Increased developer effort |
| Complex optimization patterns | Poor DX |

A first-class granular subscription model would allow React to optimize rendering behavior at a more precise level with minimal developer involvement.

## Detailed Design

`useGranular` tracks accessed values during selector execution and subscribes components to only those paths. A dependency graph stores relationships between state keys and render subscriptions.

### Example API

```js
const value = useGranular(store, s => s.deep.path.to.value);
```

### Internal Subscription Mapping Example

```
{
  "user.name": [component1, component4],
  "theme": [component2]
}
```

### Update Behavior

- When updates occur, the dependency graph locates changed paths.
- Only components subscribed to those paths re-render.
- Compatible with `useSyncExternalStore` and concurrent rendering.
- Selector evaluation is memoized, with cached subscriptions.

## Why This Design

| Existing Tool | Limitation |
|---------------|------------|
| Context | Consumers always re-render |
| Redux | Requires handcrafted selectors + memoization |
| Zustand | Granular only per top-level key |
| Jotai / Recoil | Requires splitting atoms manually |
| Signals libraries | Non-idiomatic React model |

This proposal maintains React idioms while enabling fine-grained reactivity similar to signals architectures.

## Drawbacks

- Adds complexity to internal dependency tracking.
- Must avoid overhead outweighing performance improvement in small trees.
- Requires clearly defined behavior around deeply mutable updates.

## Alternatives Considered

- Continue current manual optimization approaches.
- Depend on external signals-based ecosystems.
- Manually split state into many independent contexts.
- Compile-time reactivity via React Forget (separate effort).
- Plugin-only architecture rather than core integration.

## Adoption Strategy

- No breaking change.
- May begin as experimental / opt-in API.
- Gradual adoption in large codebases.
- Compatible with synchronous, concurrent, and server component execution boundaries.

## How We Teach This

Documentation improvements:
- Updating "State Management" and "Optimizing Re-renders" sections
- Demonstrating performance comparison between granular and full render
- Showing migration from Context-based state to granular subscription usage

```js
const name = useGranular(store, s => s.user.name);
```

## Unresolved Questions

- Should this ship in React core or as an official companion package?
- Should mutable state structures be formally supported?
- Should DevTools expose subscription paths for debugging?

## Conclusion

This RFC proposes a selector-based granular subscription mechanism for React applications, improving rendering efficiency and reducing developer effort needed to manage performance. It provides clear performance benefits with minimal API footprint and aligns with modern performance expectations for concurrent rendering.

## Request for Community and Team Feedback

- Is this direction aligned with React’s performance roadmap?
- Should this begin as an experiment?
- Preferred shape of the initial API surface?
