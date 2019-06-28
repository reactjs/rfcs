- Start Date: 2019-06-27
- RFC PR:
- React Issue:

# Summary

This RFC describes a new API, currently envisioned as a new hook, for allowing users to make a selection from a context value instead of the value itself. If a context value changes but the selection does not the host component would not update.

This is a new API and would likely remove the need for observedBits, an as-of-yet unreleased bailout mechanism for existing readContext APIs

For performance and consistency reasons this API would rely on changes to context propagation to make it lazier. [See RFC for lazy context propagation](https://github.com/reactjs/rfcs/pull/118)


# Basic example

```
let Context = React.createContext(‘’)

let App = ({ index, string }) => {
  return (
    <Context.Provider value={string}>
      <Foo index={index} />
    </Context.Provider>

}

let Foo = React.memo(({ index }) => {
  let selector = React.useCallback(s => s.substring(0, index), [index])
  let selection = React.useContextSelector(Context, selector)
  return <span>{selection}</span>
})

// Foo renders (mount) and selector is called during Foo’s render: “abcd”
ReactRenderer.render(<App index={4} string=”abcdefg” />)

// Foo renders (props update) and selector is called during Foo’s render: “abcde”
ReactRenderer.render(<App index={5} string=”abcdefg” />)

// Foo does not render (memo props same), selector is called before Foo bails out (selection same): “abcde”
ReactRenderer.render(<App index={5} string=”abcdef*” />)

// Foo renders (selection update), selector is called before Foo’s render and the result is memoized and returned again during that render: “a*cde”
ReactRenderer.render(<App index={5} string=”a*cdef*” />)

// Foo renders (props update), selector is only called during Foo’s render even though the context value also changed: “a**d”
ReactRenderer.render(<App index={4} string=”a**def*” />)

```


# Motivation

Many React applications rely on external values that change in time. The most obvious example I can think of is redux but there are plenty of other cases such as Backbone models, navigation state, game engine state machines, or any other value object that has a subscription mechanism.

External values that need to be observed in React applications have historically relied on subscription patterns or Proxy access traps to trigger updates in response to changing values. With Concurrent React however this kind of integration is going to be more challenging given updates of different priorities can be executed out of order. Many libraries today are totally unprepared for this because they read the ‘current’ state of these external values in render methods. If this value is unaware of render yielding and higher priority updates there is no current way to let them know to return an ‘old’ value temporarily while the yielded-to update processes

The modern Context API is fully integrated with Concurrent React update scheduling and would be a perfect vehicle to ‘expose’ a snapshot of the value for a given update to all descendent children. This was tried in fact with `react-redux@6`, however it performed poorly in many cases. The root cause of this poor performance is the fact that anytime a context value changes any readers of that context were updated. There was (and remains) no public optimization path. The private observedBits optimization even if it were public is not suitable for a large class of use cases, in particular cases where the context provider doesn’t know how to scope the update into arbitrary groups.

So while Context checks all the boxes for Concurrent React and will continue to support new features as they are added, such as Suspense, it is not suitable in general as a mechanism for exposing arbitrary external values to a React application.

In addition to the above problem, many applications let react components ‘register’ selectors or other access functions that depend on that components props. There are timing issues when these selectors are run ‘too early’ and are about to be changed when new props arrive which can lead to unexpected errors (zombie-child problem in react redux hooks) and wasted work (old selector ran but it needs to run again with new props, or this tree is going to unmount because of a parent that was updated so the selector never needed to be run in the first place)

If we can integrate the concept of selectors into the Context feature directly we can solve both of these problems. If a selector is run with a new context value but the selection doesn’t change we can avoid scheduling work for that component drastically increasing the incidence of update bailouts. If there is a lazy context propagation in place, selectors can be run only if we would otherwise call `bailoutOnAlreadyFinishedWork`. This means we would never run them on soon-to-be-unmounted trees, or when props are updating for other reasons. Since props will always have an opportunity to change *before* running the selector we never have a situation where the selector will be invalidated but a different part of the React update

A third benefit of this API is it would eliminate a large surface area of coordinating logic between external subscriptions and the react lifecycle. A hooks only react-redux implementation could be written in roughly 30 lines of easily understandable idiomatic react code.

It is important to recognize that while selectors can be more ‘expensive’ to run than statically checking context dependencies, including the changedBits api, there really isn’t any extra work happening. If a component updates on a context change that context value is processed in some way. The selector is implicitly embedded in the component to get to the part that is useful. By putting it into useContextSelector we’re moving where the code is run but not meaningfully increasing how much or how expensive that code is to run. This means that even with poorly constructed selectors that update all the time (new value on every call) the worst case performance is today’s status quo where the component re-renders on every change.

In summary, adding this useContextSelector API would solve external state subscription incompatibilities with Concurrent React, eliminate a lot of complexity and code size in userland libraries, make almost any Context-using app the same speed or faster, and provide users with a more ergonomic alternative to the observedBits bailout optimization.

# Detailed design

To achieve the efficiency and consistency goals of calling selectors on context value changes this RFC expects that the Lazy Context Propagation RFC has been accepted and it’s changes or meaningfully equivalent changes are in place

## Add selectFromContext (ReactFiberNewContext)
Implement a new function `selectFromContext`. This function will parallel `readContext` but will receive a selector and will setup a selector based context dependency. It will also call the selector with the appropriate `_currentValue` and return it

`selectFromContext` dependencies will integrate with `readContext` dependencies so users can mix and match useContext and useContextSelector hooks without problems

## Modify context dependency checking (ReactFiberNewContexts)
Today if a context dependency has an intersecting `observedBits & changedBits` the fiber is marked for an update. This logic will be modified as such

1. Continue to check `observedBit & changedBits`
  1. If observed update, check if context dependency has a selector
    1. If no selector, schedule update
    1. If selector is found, call it with the current context value
      1. If the selection is new, schedule update
      1. If the selection is the same as the previous selection, continue context dependency search

## Implement `useContextSelector` hook (ReactFiberHooks)
Implement a new hook `useContextSelector` of type `(ReactContext<T>, T => S) => S`

On hook mount we will construct a memoizing wrapper for the selector which will only recompute the selection if the context value has changed. It will then call `selectFromContext` to register the dependency and get back the selection.

The hook will store `[context, selector, memoizing wrapper]` as memoizedState

On hook update we check if the context or selector has changed (using memoizedState). If they have changed we recreate the memoizing wrapper and the hook’s memoizedState (more or less copying the mount behavior). If they have not changed we call `selectFromContext` from the previous memoizing wrapper


# Drawbacks

- Context Propagation still requires at least one full react tree traversal. Current subscription mechanisms can still be more efficient (even if they are not compatible with Concurrent React).


# Alternatives

- [variation] useContext could be overloaded to take a selector in the second arg position to avoid creating a new hook name. However this is not great from a type perspective since the return value is unspecified (sometimes it is the whole context and other times it is a selection type)
- [variation] Support a deps argument for the selector to have parity with other first-party hooks
- [variation] A more ambitious approach would be to implement `useContexts([C1, C2, …, Cn], (c1, c2, …, cn) => ...)` effectively you can take arbitrary context inputs and do a selection across all of them. That would allow new usage patterns especially for breaking up states and provide a generalized framework for subscribing to ‘computations’ rather than values. Implementation-wise this is not really any more complicated, though this API may be unnecessary for most cases
- Expose another mechanism for external values to use to serve up previous values if react is doing a higher priority update. This wouldn’t help with certain classes of problems such as “zombie-child” in react-redux and it would not really decrease the size of userland libraries but it could make things more Concurrent React ready
- Provide an alternative bailout that can be used inside hooks. I hate to suggest this one because it mimics shouldComponentUpdate which is really problematic and non compositional as a hook.


# Adoption strategy

This is a new hook. No existing APIs are modified so it is opt-in only

# How we teach this

If contexts are updating too often or you have many consumers that only care about portions of a context then replace your useContext instances with useContextSelector to opt out of re-renders

```
let computeValueFromContext = …


// starting with this today
function Component() {
  let context = useContext(Context)
  let whatIReallyNeed = computeValueFromContext(context)
  // … rest of function
}

// replace it with this...
function Component() {
  let whatIReallyNeed = useContextSelector(Context, computeValueFromContext)
  // … rest of function
}
```

Documentation on the React site should be updated to instruct users on the existence and use of this hook

# Unresolved questions

- Running a selector during propagation can error. Should we catch it, mark the fiber for update and let the error throw during work?
- Should deps be the last argument to this hook? It would bring parity with other first party hooks and make inline selectors shorter to write. The exhaustive deps lint rule can support this hook
- Should the multi-context variation be pursued? It is even more powerful if slightly more complicated.
- How do we motivate this API change when it relies on a separate RFC being adopted first? :(
