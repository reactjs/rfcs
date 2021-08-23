- Start Date: 2019-06-19
- RFC PR: https://github.com/reactjs/rfcs/pull/118
- React Issue: https://github.com/facebook/react/pull/20890

# Summary

This RFC describes a new lazy context propagation implementation which uses the work React is already going to do when possible, before resorting to explicit context dependency searches. This results in improved performance characteristics for some use cases. Additionally it may enable new kinds of context update bailouts to be implemented.

It does not modify the public API of React in any way.

# Motivation

The eager context propagation in place today is executed per ContextProvider. When a ContextProvider is updating to a new value every constituent fiber has its context dependencies checked against the updating Context and work is scheduled if necessary.

This approach has two direct behaviors that can be optimized. First, for any given render, each ContextProvider with a new value will result in the constituent sub-tree for that ContextProvider to be visited. In a simplified scenario, with ContextProviders at the top of the tree, if `N` ContextProviders are updating then the fiber tree will be traversed during propagation exactly `N` times.

The second optimization opportunity is that, during an update, portions of the tree may be unmounted. The eager propagation visits these soon-to-be-unmounted sub-trees since it can’t know yet they are going to be removed causing wasted work walking fibers and checking context dependencies.

The lazy propagation described in this RFC delays doing any context dependency checking and look-ahead propagation except when React would otherwise bail out of work on this fiber or its child fibers. It also checks all contexts simultaneously. These two changes allow for propagation tree traversal to be somewhere between 0 and 1 times (only parts of the tree will be visited during propagation and only once regardless of the number of contexts with changed bits). In addition propagation will only happen when no more scheduled work is found for a given sub-tree. Since unmounting is the result of changes during render, unmounts will now happen before context propagation begins.

In addition to direct optimizations, the lazy propagation will allow for some interesting new APIs. For instance a new invariant exists where if propagation has begun we know the child tree would otherwise have bailed out of updates (the workInProgress sub-tree would be the current sub-tree due to structural sharing). In this environment we can do more expensive and dynamic checking of context dependencies without fear that other work will invalidate these computations. In particular, rather than having a statically determined update bailout based on `observedBits` it would be possible to do a dynamic check running an arbitrary function receiving the context value to determine if a bailout should happen. It can be dynamic here because we know the result will not be invalidated by other work because know no other work has been scheduled. This idea will be explored more in a separate RFC soon.


# Detailed design

Imagine React did not have any kind of bailout system in place during beginWork. When context values change there would be no need to propagate them because the entire tree would re-render and each context reading component would read the latest value afresh.

When reintroducing the idea of bailing out of work we would now have a problem since some work would be skipped. This proposal essentially boils down to avoiding bailouts in beginWork if the fiber under consideration depends on a changed context value.

## Changes to work bailouts (ReactFiberBeginWork)

There are actually 3 distinct bailout classes that need be handled

### Work begins on fiber; no update scheduled

Before calling an update*Component function in beginWork, if new and old props are equal and updateExpirationTime < renderExpirationTime the update step is skipped. To avoid this bailout when context dependencies have changed we need to check them before entering this branch and reset the updateExpirationTime if necessary.

### Work begins on fiber; fiber type has built-in bailouts

Memo Components and Class Components with shouldComponentUpdate offer bailout mechanisms that are executed during the update step of a component. Even if workInProgress props are different from current props the fiber may bail out of work. In these cases, before choosing to bail out of work, context dependencies should be evaluated and if changes are observed the bailout should be avoided.

### Work is bailing out on fiber; child fibers will be skipped

During a bailout of a fiber, this fiber’s childExpirationTime is checked to see if any descendents have work scheduled and if not, work skips over child fibers (reusing the current fibers from the previously committed tree). When this is about to happen a tree-walking propagation algorithm will run, nearly identical to the existing propagation algorithm ensuring we do not miss any necessary context updates.


### Summary of changes
- In `bailoutOnAlreadyFinishedWork`, implement an invariant that guarantees context dependencies were checked prior to initiation. It will be a bug to bail out of work without some explicit context dependency check or where it is known the fiber type cannot have context dependencies
- Implement a `canBailout` function which does a check (if necessary) and only returns true if it is safe to bail out of work on this fiber. Calling this would satisfy the invariant above
  - Checks context dependencies if needed
  - Can be called multiple times (bailouts can happen at more than one point through the steps of beginWork), but will only check dependencies the first time
- During `bailoutOnAlreadyFinishedWork`, run `propagateContexts` before checking childExpirationTime to determine if work should continue deeper or return to this fiber’s sibling

## Preventing more than one check per fiber

The context propagation when it does run will cause context dependencies to be checked on fibers that have not yet had work begun. If new updates are scheduled work will then begin on these fibers. To avoid rechecking context dependencies a second time we can add a propagationSigil to every Fiber. Whenever a Provider is pushed (beginWork for a ContextProvider component) we will create a new module global `propagationSigil` (an empty object) which we can assign to fibers that have had their context dependencies checked. Later if another check starts and the propagationSigil is the same the dependency check can be skipped. When a new Provider is pushed the sigil will change ensuring we re-run checks given the new aggregate context state has been modified.

## Early Propagation Bailouts

If we know there are no contexts with changedBits we can skip all propagation. a new module global `propagationHasChangedBits` will be set as Providers are pushed and popped. it will be false when we know there are no Providers with changedBits

## Changes to ContextProviders

- Stop triggering a propagation on beginWork
- During pushProvider
  - set a `_currentChangedBits` value
  - create and set a new `propagationSigil`
  - declare whether any pushed contexts have changed bits in via `propagationHasChangedBits` boolean.
- During popProvider
  - Restore previous `_currentChangedBits` value
  - Restore previous `propagationSigil`
  - Restore previous `propagationHasChangedBits`

## Changes to Propagation Algorithm

- Break out context dependency checking into a separate method that can be called from beginWork without having to run the full propagation algorithm. It will read `_currentChangedBits` from each context instead of taking `changedBits` as an argument
- Bail out of propagation if no contexts have changed bits (see `propagationHasChangedBits`)
- Will consider any context dependency that has intersecting changed bits as requiring an update (checking all contexts together)
- Does not go deeper on fibers with `expirationTime | childExpirationTime >= renderExpirationTime`. These will be visited during beginWork and do not require any kind of look-ahead context propagation.
- Schedules work on any ContextProviders it visits. This is necessary because this Provider might mask a currently changed value. We can no longer propagate *through* any Providers
- Set current `propagationSigil` on fiber to avoid rechecking during `beginWork`


# Drawbacks

- This implementation is more complex & the performance benefits are not yet quantified.
- Adding new bailouts would now be more complicated because context needs to be considered


# Alternatives

Keep existing propagation algorithm

# Adoption strategy

Adoption will be automatic, there are no public api changes.

# How we teach this

No teaching necessary

# Unresolved questions

Suspense will need special handling (in particular server side rendered suspense components). I do not understand this feature well enough yet to know how to properly integrate it with Lazy context propagation but I believe it can be done.
