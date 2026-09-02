- Start Date: 2026-09-02
- RFC PR: (leave this empty)
- React Issue: https://github.com/facebook/react/issues/25279

# Summary

This RFC proposes an experimental, renderer-aware primitive for virtualized
rendering. It would allow list implementations to identify visible and
offscreen work, schedule the two classes differently, and preserve React state
when offscreen items leave the host tree.

The proposal does not attempt to make browser scrolling, item measurement, or
list layout part of React Core. Those concerns remain in a replaceable
userland or renderer-specific list implementation.

# Basic example

The following is an API sketch, not a final API:

```jsx
const {
  visibleRange,
  overscanRange,
  totalSize,
  measureItem,
} = useVirtualizer({
  count: rows.length,
  estimateSize: () => 40,
  overscan: 8,
  viewport,
});

return (
  <VirtualizedBoundary
    visibleRange={visibleRange}
    overscanRange={overscanRange}
    getItemKey={index => rows[index].id}
    renderItem={index => (
      <Row
        data-index={index}
        ref={measureItem}
        row={rows[index]}
      />
    )}
  />
);
```

The important distinction is that `useVirtualizer` computes a range from
host-provided viewport and measurement information, while
`VirtualizedBoundary` gives React a lifecycle and scheduling boundary for the
result. The names and exact split are intentionally open for review.

# Motivation

Rendering a large list by creating every row at once can produce a slow
initial mount, long update tasks, high memory usage, and scroll jank. This is a
common enough problem that React applications routinely install a separate
virtualization package. A prior React issue requested a built-in
`VirtualizedList` specifically because existing packages can be difficult to
use with dynamic row sizes, window scrolling, and mobile browsers:

https://github.com/facebook/react/issues/25279

React Native already has host-specific list primitives such as
`VirtualizedList` and `FlatList`. Web applications and custom renderers have
different measurement and scrolling constraints, so a single DOM-oriented
component in `react` would not be a universal solution.

React has also explored Offscreen rendering as a lower-level capability for
virtualized lists. The React team described the desired direction as allowing
list frameworks to render additional rows at a lower priority and preserve
state for content that is not currently visible:

https://react.dev/blog/2023/03/22/react-labs-what-we-have-been-working-on-march-2023

The goal of this proposal is therefore narrower than “React should own every
list implementation.” It is to identify the minimum React contract needed for
list implementations to integrate virtualization with concurrent rendering,
Suspense, effects, and state preservation.

# Detailed design

## Terminology

* A **visible range** contains items that should be present for the current
  viewport.
* An **overscan range** contains nearby items that may be prepared ahead of
  the viewport.
* A **distant range** contains items outside the visible and overscan ranges.
* A **virtualized boundary** is a React-managed boundary that coordinates the
  lifecycle and priority of those ranges.

## Responsibilities of the list implementation

The list implementation owns:

* Reading scroll offset and viewport dimensions.
* Measuring fixed or dynamic item sizes.
* Computing visible and overscan ranges.
* Choosing item keys.
* Mapping items to layout positions.
* Implementing `scrollToIndex` and other layout-specific operations.
* Providing host-specific accessibility behavior.

The reference prototype in the companion React fork demonstrates this layer.
It uses a Fenwick tree so measured-size updates are `O(log n)` and range
lookups do not materialize the entire list. Its tests cover a 1,000,000-item
case while producing only the visible range plus overscan.

## Responsibilities of React

React would own the boundary semantics:

1. Visible work is scheduled at normal priority.
2. Overscan work may be rendered at a lower priority.
3. Distant work can be deactivated or detached from the host environment.
4. The boundary defines whether component state is retained when host
   instances are detached.
5. Effects follow documented visibility and detachment semantics.
6. Suspense and transitions remain consistent when ranges change during an
   interrupted render.
7. Host renderers can provide the representation of hidden or detached
   instances.

One possible implementation could extend the semantics of the existing
`Activity`/Offscreen machinery. Another could introduce a separate experimental
boundary. The RFC does not assume that the current `Activity` API is sufficient
or that either name should become public.

## State and effects

The most important semantic question is whether an item outside the host tree
can retain React state without retaining all of its host instances. If state is
retained, the API must specify:

* whether `useEffect` cleanup runs when an item becomes distant;
* whether layout effects are disconnected while the item is hidden;
* what happens when an item returns with changed props;
* how keys identify retained state; and
* how memory is bounded when a user scrolls through a very large dataset.

The default should be conservative and predictable. A list implementation
should be able to opt into state retention only when it can provide stable keys
and acceptable memory behavior.

## Cross-renderer behavior

The primitive must not require a DOM element, `ResizeObserver`, or browser
scroll events. A renderer may represent an inactive subtree by hiding it,
detaching host instances, or using another safe mechanism. The public contract
should describe React lifecycle and priority semantics, while the host config
defines how those semantics are realized.

## Server rendering and hydration

The RFC needs a defined policy for the initial server viewport. At minimum, a
list should be able to render a deterministic initial range and hydrate it
without mismatches. Streaming and Suspense must not cause the server and client
to disagree about item keys or retained state.

# Drawbacks

* A new primitive increases React's already small public API surface.
* Virtualization can be implemented in userland today, so React-specific value
  must be demonstrated rather than assumed.
* State retention for detached items could increase memory usage.
* Effect semantics may be surprising if they differ from ordinary unmounting.
* Accessibility and focus management remain difficult and cannot be hidden by a
  generic primitive.
* Supporting DOM, React Native, and custom renderers may add substantial
  implementation and testing cost.
* A built-in range algorithm could become a maintenance burden and constrain
  future list-library designs.

# Alternatives

## Continue using userland packages

This is the simplest option and remains a valid outcome. Libraries such as
`react-window`, `react-virtual`, and `react-virtuoso` already provide useful
solutions. The cost is duplicated integration work and limited access to
React's scheduling and lifecycle internals.

## Add a DOM-specific `VirtualizedList` component

This could offer a simpler user experience on the web, but it would not be a
universal React solution. It would need separate behavior for React Native,
custom renderers, dynamic layout systems, accessibility, and server rendering.

## Add only a `useVirtualizer` Hook

A Hook can package range calculation, but it cannot by itself define how React
retains state, detaches host instances, or schedules offscreen work. The
prototype uses this approach to validate the algorithm, but the core proposal
needs a renderer-aware boundary if React is to provide additional value.

## Use existing `Activity` only

`Activity` may already provide part of the required lifecycle behavior. The
prototype and RFC should test whether it can support virtualized lists without
new semantics. If it can, a separate API may not be necessary.

# Adoption strategy

This would begin as an experimental API and would not change existing list
behavior. A list library could progressively adopt it behind a feature check,
while existing applications continue using their current virtualization
package.

The likely adoption sequence is:

1. Validate the design with a userland reference implementation.
2. Add renderer and browser experiments for state, effects, hydration,
   accessibility, and memory.
3. Implement an experimental React build behind the normal release-channel
   and feature-flag mechanisms.
4. Test the primitive in a small number of list libraries and renderers.
5. Document lifecycle behavior and migration guidance.
6. Consider a stable API only after real-world feedback.

No codemod should be required because this is additive. React should not ship a
complete list package as part of this proposal unless later evidence shows that
the range and layout layer can be specified portably.

# How we teach this

The central teaching point should be that virtualization has two layers:

* a list layer that knows about data, layout, viewport, measurement, and
  accessibility; and
* a React layer that schedules and manages the lifecycle of offscreen work.

The documentation should begin with a complete userland list example, then
explain when a list library can use the React primitive. It should clearly
distinguish “not currently visible” from “unmounted,” because state and effect
behavior may differ.

# Unresolved questions

1. Is a new public primitive necessary, or can `Activity` be extended safely?
2. Should the range calculator be part of React, `react-dom`, or remain in
   userland?
3. Can React retain state for distant items while bounding memory?
4. What effect semantics are least surprising for hidden and detached items?
5. How should focus and accessibility APIs behave when the focused item leaves
   the visible range?
6. What minimum host-config contract supports DOM, React Native, and custom
   renderers?
7. How should server rendering choose and hydrate the initial range?
8. What measurements would demonstrate that React Core provides meaningful
   value over existing packages?
