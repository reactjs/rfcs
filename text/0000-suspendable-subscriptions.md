- Start Date: 2019-10-30
- RFC PR: (leave this empty)
- React Issue: (leave this empty)


# Summary

Adds two new features to `useSubscription`:

- Support an `isSuspended()` option, allowing subscriptions to suspend until their initial value is ready.
- Allow it to accept an array of subscriptions, and suspend until all subscriptions are ready.

Together, these changes would allow a component to fetch data from multiple independent data sources *without* encountering waterfalls.


# Basic example

```js
import { useSubscription } from "use-subscription"
import { cache, subscribe } from "data-source"

function useDataSource(id) {
  return useMemo(() => ({
    getCurrentValue() {
      return cache[id] || { pending: true }
    },
    isSuspended() {
      return !cache[id]
    },
    subscribe(callback) {
      return subscribe(id, callback)
    },
  }), [id])
}

function Page() {
  // The component will suspend until all subscriptions are ready
  const [
    value1,
    value2
  ] = useSubscription([
    useDataSource('a'),
    useDataSource('b')
  ])

  // Rendered output goes here...
}

```


# Motivation

Concurrent Mode now enables apps to suspend a component until its initial data is available. Given time, it's likely that many different libraries will provide hooks that make use of this feature.

For example, you can imagine a developer may want to use both a GraphQL `useQuery` hook, along with ZEIT's new [useSWR](https://github.com/zeit/swr) hook:

```js
function Page() {
  const todos = useSWR('/api/todos', fetch)
  const user = useQuery(USER_QUERY)
}
```

Say that both hooks are configured to suspend until the requested data is ready, and they both automatically fetch their required data when they're first called. Given that this is the case, the `useQuery` hook cannot start fetching its data until the `useSWR` hook's fetch has completed. **The component has a fetch waterfall**.

The issue here may become easier to grasp through an analogy with promises. Imagine that components are async functions, and hooks that suspend are async functions with an `await` in front of them:

```js
async function Page() {
  const todos = await useSWR('/api/todos', fetch)
  const user = await useQuery(USER_QUERY)
}
```

This demonstrates why the fetch waterfall is occurring: `useQuery` is not executed at all until the final result of `useSWR` is available. The analogy also suggests a possible solution: you could wait for both results simultaneously using `Promise.all()`:

```js
async function Page() {
  const [
    todos,
    user
  ] = await Promise.all([
    useSWR('/api/todos', fetch),
    useQuery(USER_QUERY)
  ])
}
```

The analogy breaks down here; components and hooks are not *really* async functions. However, a similar result could be achieved through suspendable subscriptions. If `useSWR`, `useQuery`, etc. were each to return a suspendable subscription object following a common format, then React's `useSubscription` hook could take on a similar role to `Promise.all`:

```js
function Page() {
  const [
    todos,
    user
  ] = useSubscription([
    useSWR('/api/todos', fetch),
    useQuery(USER_QUERY)
  ])
}
```

While this approach does not *guarantee* elimination of fetch waterfalls, it does provide a simple approach that would be sufficient in many common situations. Along the way, it'd create a common API that data sources can conform to, easing the task of correctly pulling data into a Concurrent Mode React app, and simplifying the task of integrating multiple independent data sources. Given adoption by libraries like Relay, Apollo and `useSWR`, consuming data within React applications would become simpler, more reliable *and* more performant than ever.


# Detailed design

This RFC proposes two additions to the existing `useSubscription` API:

1. An optional `isSuspended()` property for subscriptions
2. Ability to an array of subscriptions


## `isSuspended()`

When a component first subscribes to a data source, it's often the case that a placeholder will be used for the initial value, e.g.

```js
{
  pending: true
  value: undefined,
}
```

For data consumption hooks that are built with a specific source in mind, it's possible to infer whether values are placeholders by looking at the values themselves.

```js
const shouldSuspend = value.pending
```

However, given that `useSubscription` needs to work for *any* type of data -- including strings and `undefined` -- it's not possible to tell whether a value is a placeholder from the value itself. Instead, this information needs to be available separately. I suggest that the simplest approach is to allow subscriptions to specify this with an optional `isSuspended()` method.

```typescript
interface SuspendableSubscription<T> {
  getCurrentValue: () => T,

  // If this exists, and returns true, then the current value should be
  // treated as a placeholder value.
  isSuspended?: () => boolean

  subscribe: (callback: () => void) => void,
}
```

When `isSuspended` exists, it must be a function that returns a boolean. A return of `true` indicates that the data is a placeholder (but may contain a partial result). A return of `false` indicates that *all* requested information is available.

For backwards compatibility, when `isSuspended` does not exist, it would be assumed that the subscription is *not* suspended.

Whenever the value returned by `isSuspended` changes, a subscription should call its `subscribe` callbacks. This allows `useSubscription` to create and throw a promise that resolves when `isSuspended` becomes `false`.


## Subscription arrays

When `useSubscribe` receives an *array* of subscriptions as its first argument, it should return an array of the current values of each subscription. This is similar in concept and usage to `Promise.all`:

```js
const [value1, value2] = useSubscription([subscription1, subscription2])
```

When *any* subscription is suspended, `useSubscription` should throw a Promise that resolves once *all* subscriptions are ready.


## When to release subscriptions

Some data sources may use the lack of a `subscribe()` callback as a signal that data can be purged from a cache.

```js
// Fetches and caches data
const release = subscription.subscribe(data => console.log(data))

await tenSeconds

// Purges fetched data from cache
release()
```

While this shouldn't cause any issues with a single subscription, it *may* cause issues if `subscribe` callbacks are released while waiting for a suspended subscription to be ready.

Consider the example of two suspended subscriptions, where the first resolves 10 seconds earlier than the second:

- `useSubscription` subscribes to subscriptions 1 and 2.
- subscriptions 1 and 2 both fetch the requested data, suspending until it is available.
- after 1 second, subscription 1's data is ready. `useSubscription` calls its `release()` function.
- after 5 seconds, subscription 1 purges the fetched data
- at 10 seconds from initial render, subscription 2's data is ready. The promise thrown by `useSubscription` resolves.
- React re-renders the component, but subscription 1's data has been purged from cache, so it suspends again and refetches.

In order to avoid this scenario, it's important that *all* subscriptions opened before `useSubscription` throws a promise and held open until after that promise has resolved.


## Legacy Mode support

To allow suspendable subscriptions to be used in Legacy Mode, a second argument could be passed to `useSubscription` that configures whether to use suspense.

```js
useSubscription(subscription, {
  suspend: false
})
```

To enable this, a suspended subscription's `getCurrentValue()` should always return a placeholder result.


# Drawbacks

This proposal may cause issues for any apps that already pass an `isSuspended` method to `useSubscription`.

It would also substantially increase the complexity of `useSubscription`, even though the same features could be implemented in application code. If `useSubscription` is adopted by the community, this may actually result in smaller bundle sizes on average, as it would consolidate subscription management code in a single place. However, encouraging adoption by the community would take time and effort. 

The primary benefit of suspendable subscriptions is that they allow integration of multiple independent data sources in a performant manner -- but this is a double edged sword. Without support of 3rd party library developers, it's unlikely that `useSubscription` will see much use. And while `useSubscription` should be reasonably easy to teach *given* that there are data sources available, teaching developers to create new data sources would be a much harder problem.

One thing that was not immediately obvious to me was that this proposal does *not* add any requirements for memoization of subscription objects -- at least over the current version of `useSubscription`. This is due to the fact that a component suspending on its initial render cannot store any state, and thus is unable to memoize anything anyway.


# Alternatives

It's possible for libraries for data sources to each include their own hook along these lines. Similarly, its possible to avoid libraries for data sources and just fetch data with useEffect/useReducer.

The main advantage to having this within React itself would be:

- Allowing multiple independent data sources to work together without waterfalls
- Providing a reference implementation of complex subscription logic
- Simplifying the task of correctly consuming a data source in Concurrent Mode


# Adoption strategy

The only situation in which this will be a breaking change is when a developer already provides an `isSuspended()` method to `useSubscription`. This should be a relatively rare occurrence, so there shouldn't be any need for code mods.

However, given that subscription objects themselves can be hard to create, the main issue that adoption faces would be the lack of things to subscribe to in the first place. Coordination with other projects/libraries would be key here.


# How we teach this

Teaching this can be broken into two distinct parts:

1. Teaching how to `useSubscription`, given you already have a subscription object
2. Teaching how to create subscription objects

The first of these should be relatively easy, and could take the form of a documentation page. It may help to have an external library for creating simple subscriptions, ala react-cache. The main hurdle in teaching `useSubscription` would be in discouraging users from calling `useSubscription` multiple times, and encouraging them to use the array form instead.

As for teaching how to create subscription objects, there would need to be some sort of documentation on this, but it would be mainly targeted at advanced users. The exact form of this is TBD.


# Unresolved questions

- `isSuspended` vs. `isPlaceholder`, or some other name. 
- Should React provide some sort of basic cached subscribable, ala react-cache? This could add some significant weight, but would `useSubscription` far more accessible to beginners.
- Should this be part of React itself, or stay in the `use-subscription` package?
- Does `useSubscription` need to do anything extra to handle errors, or can subscriptions just throw errors from `getCurrentValue()`?