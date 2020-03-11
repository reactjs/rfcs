- 2020-02-27
- RFC PR:
- React Issue:

# Summary

This RFC proposes a change to the fiber work implementation to defer creating workInProgress fibers until and unless it is certain we need to. It builds on ideas initially outlined in #118 and #119 where context propagation was done lazily. This general idea is extended to all kinds of work and is summarized as follows.

When we can bail out of updating a given component but need to go deeper into the fiber tree to potentially do additional updates we no longer create a workInProgress fiber but instead start work on the current fiber in "speculative mode". If a deeper updates all bail out then we can completely avoid a large amount of work React previously had to do. If a deeper update results in a meaningful change (new children, or new effects) we reify the workInProgress fibers that would have been created and continue work normally.

This update will make bailouts less expensive which means more work can be done during the work step of the fiber itself without paying a high cost. Here are some things that this might enable

* context selectors: only update a component when the used part of a context value changes
* useReducer bailouts: only do a single reducer call even if the reducer function changes before
* faster updates when returning memoized children

# Basic example

nothing necessarily changes in the public API but here are some potential changes that could be built on top or elided by testing react implementation details

## context selector

similar to #119. support a second argument that is a function which takes the context value and returns anything

```
// app would only be re-rendered if theThingICareAbout changed even if other parts of MyContext changed
let App = () => {
  let theThingICareAbout = React.useContext(MyContext, ctx => ctx.theThingICareAbout);
  return <span>{theThingICareAbout}</span>
}
```

## single pass useReducer bailouts
imagine the following component
```
let Adder = ({ additional }) => {
  let addingReducer = (previousValue, plus) => previousValue + plus + additional;
  let [val, dispatch] = React.useReducer(addingReducer, 0);
  return <span>{val}</span>
}
```
notice the reducer is not memoized and not static. it will have a different identity on every render.

if `additional` is zero and we `dispatch(1)` then reducer will return `previousValue + 1` and work will be scheduled.

when the component renders the reducer identity is different (because `additional` could have changed) so we need to recompute the new state.

With speculative work we would not actually eagerly compute the state in an attempt to bail out. work would be scheduled and when this fiber began the render the bailout would occur. but only if props did not change. this way we guarantee we only ever run the reducer once per update and we do it with the latest version of the reducer function.

This means memoizing your reducer isn't important anymore. you can drop a common `useCallback` usage now b/c dynamic or static reducers have the same performance characteristics

## faster updates when returning memoized children

imagine the following component deep within the component tree

```
// static elements
let even = <span>even</span>
let odd = <span>odd</span>

// a component to be rendered deep in the React tree
let MyComponent = () => {
  let [state, setState] = React.useState(0);
  return state % 2 === 0 ? even : odd;
}
```

if we `setState(1)` followed by a `setState(3)` the rendered output isn't going to change (odd in both cases). in today's React the ancestor tree of this fiber needs to have workInProgress fibers created just to figure out that nothing meaningful happened here. In Speculative mode we avoid expensive fiber creation and see that the second update did not have any side effects or updated children and so it avoids creating the workInProgress tree making it significantly faster.

Please note that if the component had the following extra line
```
let MyComponent = () => {
  //...
  React.useEffect(() => console.log('new state'), [state]);
  //...
}
```
then these updates would still result in a workInProgress tree because an effect was emitted when the state changed

# Motivation

There are multiple motivations.

First, this feature has the opportunity to make React faster by default in many usage patters without slowing down every other case by meaningful amounts.

Additionally, with Concurrent mode, bridging mutable values from outside React into React is challenging and Context is one effective way at implementing this integration. however Context is generally felt to be too slow for many use cases (for example Redux tried and failed to use it). context selectors will likely be fast enough for almost all cases.

Context in general regardless of Concurrent mode could benefit from being faster and speculative work is one way to make it so.

Finally there are just general performance speed and memory benefits to only creating workInProgress when absolutely required.

# Detailed design

The implementation centers around bailouts. When we bailout of work on a fiber but have child fibers to visit we will no longer automatically clone them. instead we will enter "Speculative mode" and start doing work on the current fiber rather than the workInProgress.

## Bailing out before "rendering"

During work, when in speculative mode, certain assumptions can be made. 1) new and old props are identical. no selectors, reducers, or other dynamic arguments to hooks and other state updating functions have been modified from the previously committed fiber.

when a current fiber has an updateExpirationTime set (and we are in speculative mode) we need to attempt to bail out of that update.

For many components speculative mode implies no updates were possible since they have no inherent update mechanism other than props (ContextProviders and HostComponents for example). in these cases we can in theory bailout without any extra computation

For Function Components this bailout involves traversing the hooks linked list and determining if any contexts or reducers have changed values.

For Class Components this would be similar but would use relevant context reader and state updater apis.

## Bailing out after "rendering" (Function Components)

If we cannot bailout before rendering we create a detached workInProgress fiber to do work on. At this point it is not connected to the workInProgress root in any way because it's ancestor fibers still come from the current tree.

After rendering the Function component we can still potentially avoid reifying the workInProgress tree by attempting another kind of bailout. If we look at the new children returned from rendering and they referentially match the current children AND if we did not push any effects (no use[Layout]Effect calls had new dependencies) we can essentially ignore the render.

This feature requires keeping track of referential identify of newChildren.

This feature has consequences that I hope might be considered relying on React implementation details. for instance if the Function component modifies a variable outside the function scope without using an effect we might ignore the render even though the function call did result in an observable effect.

## Changes to readContext and context selectors

Today all context readers create a dependency on the reading fiber but do not memoize the context value. for instance useContext does not actually create a "hook" object the way every other hook does. this is possible because bailouts for context updates are resolved eagerly by the propagator (ContextProvider) and not during the render cycle when doing work on the reader. This would have to change as we need to be able to decide during the render if any possible source of work would 'disallow' a bailout (when in speculative mode). To that end useContext as a hook needs to become a real hook with memoized state

Extending this to support a selector is relatively trivial and would provide all the benefits of #119 just executed in a slightly different process

## Changes to dispatchAction (part of useReducer and useState setters)

Today state updates are bailed out of eagerly if possible (no pending updates and not in render phase). this would be removed to allow for that work to execute alongside the fiber bailout process of speculative work and not eagerly in the dispatch function call.

## Class Components

I have not yet spent time figuring out how take advantage of speculative mode with Class Components but I believe it is possible. This section will be updated as thought is put into the implementation.

## Mode tracking

I've already tried a few ways of 'defining' when we're in speculative mode. the current implementation is to set a Speculative root (The last workInProgress fiber before work starts on the current tree). This works well but has some limitations. Alternatively we can explore using the mode property of the fiber to host a Speculative mode. I expect this will reduce a good bit of boolean logic and function calls to check the mode but requires more mutations on the affected fibers

## Suspense

I have not yet investigated how Suspense would be affected by this. more research incoming

# Drawbacks

I don't think there are any drawbacks to the idea of avoiding creating the workInProgress tree when we can get away with it. however there are drawbacks with specific implementations or sub-features. Here are a few

- Adding a property to fibers to support bailing out after rendering may not be worth it
- changing readContext into a memoizing hook may not be worth the memory pressure and allocation overhead
- If there are a large set of fibers which can not support speculative work (they require reification every time they render) then real-world fiber trees may not be able to take full advantage of the performance gains
- Context selectors may not be worth the memoization cost. could get much of the benefit with the "bailout after rendering" feature

# Alternatives

None that I am aware of at this time

# Adoption strategy

For the majority of this feature there will be no public api changes. If context selectors are included then we would need to teach that with additional documentation and encourage adoption where it is beneficial

# How we teach this

Generally there is not much to teach

# Unresolved questions

Support for Class components
Support for Suspense
Support for React Fresh in dev mode
