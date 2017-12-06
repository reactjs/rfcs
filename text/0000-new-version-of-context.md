* Start Date: 2017-12-05
* RFC PR: (leave this empty)
* React Issue: (leave this empty)

# Summary

Introduce a new version of context that addresses existing limitations.

# Basic example

```js
type Theme = 'light' | 'dark';
// Pass a default theme to ensure type correctness
const ThemeContext: Context<Theme> = React.createContext('light');

class ThemeToggler extends React.Component {
  state = {theme: 'light'};
  render() {
    return (
      // Pass the current context value to the Provider's `value` prop.
      // Changes are detected using strict comparison (Object.is)
      <ThemeContext.Provider value={this.state.theme}>
        <button
          onClick={() =>
            this.setState(state => ({
              theme: state.theme === 'light' ? 'dark' : 'light',
            }))
          }>
          Toggle theme
        </button>
        {this.props.children}
      </ThemeContext.Provider>
    );
  }
}

class Title extends React.Component {
  render() {
    return (
      // The Consumer uses a render prop API. Avoids conflicts in the
      // props namespace.
      <ThemeContext.Consumer>
        {theme => (
          <h1 style={{color: theme === 'light' ? '#000' : '#fff'}}>
            {this.props.children}
          </h1>
        )}
      </ThemeContext.Consumer>
    );
  }
}
```

# Motivation

Typically, data in a React application is passed top-down (parent to child) via
props. But sometimes it's useful to pass values through multiple levels of
abstraction without involving each intermediate. Examples include a locale, or a
UI theme. Many components may rely on those but you don't want to have to pass a
`locale` prop and a `theme` prop through every level of the tree.

Context in React provides a mechanism for a child component to access a value in
an ancestor component. In this document, we'll refer to the ancestor as the
**provider** and the child as the **consumer**.

## Drawbacks of the existing version of context

### shouldComponentUpdate blocks context changes

The main flaw with context today is how it interacts with
`shouldComponentUpdate`. If an intermediate component bails out using
`shouldComponentUpdate`, and there are no pending updates further down the tree,
React will assume there are no changes and reuse the entire subtree. If the
subtree contains a context consumer, the consumer will not receive the latest
context. In other words, context changes will not propagate through a component
whose `shouldComponentUpdate` return false.

`shouldComponentUpdate` is a fairly common optimization in React applications. A
shared component or open source library can't assume that applications won't use
it. In practice, this means that context alone is not a reliable way to
broadcast changes.

### Shifts complexity to user space

Today, developers circumvent the `shouldComponentUpdate` problem using
subscriptions:

* The provider acts as an event emitter. It keeps track of the most recent
  context value, and a list of subscribers to be notified whenever it changes.
* The consumer accesses the provider's event emitter using the context API.
  (This usage is fine because the event emitter itself does not change).
* The consumer registers an event listener with the provider.
* When the provider emits a change event, the consumer is notified and calls
  `setState` on itself to schedule a re-render.

Subscriptions are widely used by open source libraries such as Redux and React
Broadcast. It works, but it has some clear drawbacks:

* Not ergonomic. Given how common the context use case is, it shouldn't be so
  difficult to implement properly.
* Start-up cost. Setting up subscriptions for every consumer is costly,
  especially since they aren't used during the initial mount.
* Encourages mutation and other non-idiomatic patterns that could cause bugs in
  async mode.
* The same code ends up being duplicated by every library, increasing bundle
  sizes.

The meta problem is that the ownership and responsibility for a core feature has
been shifted from the framework to its users.

## Main goals of this proposal

* Zero cost (or close to it) for initial mount, commit, and unmount, trading-off
update cost as necessary.
* Easy-to-use API.
* Statically typable.
* Encourage async-friendly practices, like immutability.
* Discourage non-ideal practices, like event emitters and mutation.
* Eliminate duplicated complexity in userland code.

# Detailed design

Introduces new component types: `Provider` and `Consumer`.

```js
type Provider<T> = React.Component<{
  value: T,
  children?: React.Node,
}>;

type Consumer<T> = React.Component<{
  children: (value: T) => React.Node,
}>;
```

Providers and consumers come in pairs—for each provider, there is a
corresponding consumer.

A provider-consumer pair is created using `React.createContext()`:

```js
type Context<T> = {
  Provider: Provider<T>,
  Consumer: Consumer<T>,
};

interface React {
  createContext<T>(defaultValue: T): Context<T>;
}
```

`createContext` requires a default value to ensure type correctness.

Note that an arbitrary provider and arbitrary consumer cannot necessarily be
used in conjunction, even if their value type is the same. They must be the
result of the same `createContext()` call.

The provider accepts a context value as a prop. Any matching consumer in the
provider's subtree can access it, regardless of how deeply it's nested.

```js
render() {
  return (
    <Provider value={this.state.contextValue}>
      {this.props.children}
    </Provider>
  );
}
```

To update the context value, the parent re-renders and passes a different value.
Changes to context are detected using `Object.is` comparison. (Referred to as
strict comparision in this proposal, though `Object.is` is different from `===`
in some edge cases.) This is meant to encourage the use of immutable or
persistent data structures. In the typical scenario, context is updated by
calling `setState` on the provider's parent.

The consumer uses a render prop API:

```js
render() {
  return (
    <Consumer>
      {contextValue => <Child arbitraryProp={contextValue} />}
    </Consumer>
  )
}
```

Notice how in the above example, the context value can be passed to any
arbitrary prop on the child component. The advantage of the render prop API is
that we avoid clobbering the prop namespace.

If a consumer is rendered without a matching provider as its ancestor, it
receives the default value passed to `createContext`, ensuring type safety.

## Implementation

TODO.

# Drawbacks

## Relies on strict comparison of context values

This proposal uses strict (reference) comparison to detect changes to context
values. This is partly to encourage the use of immutable or persistent data
structures. But many common data sources rely on mutation. For example, certain
implementations of Flux, or even newer libraries like Relay Modern.

However, there are inherent problems with mutation when combined with async
rendering, mostly related to tearing. For architectures that rely on mutation,
developers will either decide some level of tearing is acceptable, or evolve to
better support async. Regardless, these problems are not exclusive to the
context API.

An escape hatch for libraries that rely on mutation is to clone the outer
container. (Or even just alternate between two copies.) React will detect a new
object reference and trigger a change.

## Only one provider type per consumer

The proposed API only allows for a consumer to read values from a single
provider type, unlike the current API, which allows a consumer to read from an
arbitrary number of provider types.

The solution is to use compose consumers:

```js
<FooConsumer>
  {foo => (
    <BarConsumer>
      {bar => (
        // Render using both foo and bar
        <Child foo={foo} bar={bar} />
      )}
    </BarConsumer>
  )}
</FooConsumer>;
```

Most abstractions around context already use similar patterns.

# Alternatives

## setContext

Instead of relying on reference equality to detect changes to context, we could
instead use a `setContext` API that works like `setState`. However, setting
aside the implementation overhead, this API would only be valuable when combined
with mutation, which we're specifically aiming to discourage.

## Passing context to shouldComponentUpdate

One argument is that we could avoid the `shouldComponentUpdate` problem by
passing context as an argument to that method, compare the incoming context to
the previous context, and return true if they are different. The problem is
that, unlike props or state, we have no type information. The type of the
context object depends on the component's position in the React tree. You could
perform a shallow comparion of both objects, but that only works if we assume
the values are immutable. And if we're going to assume the values are immutable,
React might as well do the comparison automatically.

## Class-based API

Instead of render props, we could use a class-based API similar to the one we
have today:

```js
class ThemeToggler extends React.Component {
  state = {theme: 'light'};
  getChildContext() {
    return this.state.theme;
  }
  render() {
    return (
      <>
        <button
          onClick={() =>
            this.setState(state => ({
              theme: state.theme === 'light' ? 'dark' : 'light',
            }))
          }>
          Toggle theme
        </button>
        {this.props.children}
      </>
    );
  }
}

class Title extends React.Component {
  static contextType = ThemeContext;
  componentDidUpdate(prevProps, prevState, prevContext) {
    if (this.context !== prevContext) {
      alert('Theme changed!');
    }
  }
  render() {
    return (
      <h1 style={{color: this.context.theme === 'light' ? '#000' : '#fff'}}>
        {this.props.children}
      </h1>
    );
  }
}
```

The advantage of this API is you'd have easier access to context inside
lifecycle methods, possibly avoiding the need for an extra component in the
tree.

However, although increasing the depth of a React tree incurs some overhead, the
advantage of using special component types is that it's faster to scan the tree
for consumers, because we can quickly skip over the other types. With the
class-based API, we'd have to check every class component, which is slightly
slower. This is likely enough to offset the cost of the extra component.

Unlike the class API, the render prop API also has the advantage of being
sufficiently different from the existing context API that we could support both
versions during a transitional period without too much confusion.

## Algebraic effects

In the past, we've considered modeling context in terms of
[algebraic effects](https://github.com/reactjs/react-basic#algebraic-effects).
Our conclusion was that memoizing effects (in order to skip over subtrees
without reevaluating them) made this approach intractable.

# How we teach this

This would be our first API that uses render props. They are an increasingly
popular pattern in third-party React libraries. However, there could be learning
curve for beginners.

Other than that, the use case is one that most React developers will be familiar
with. We'll update the context documentation to use the new API.

# Plan for adoption

Before releasing, reach out to prominent third-party library maintainers.

Initially, we will release the new API alongside the existing context API as a
minor update. The APIs are different enough to avoid much confusion.

Publish a migration guide. Make it clear that although the APIs are different,
the new version of context provides a superset of functionality. Sending down
static values is still supported.

Replacing subscription-based patterns in favor of context's built-in change
propgation may require a larger effort. However, as a first step, developers can
migrate to the new API without abandoning subscriptions. Or, they can keep
using subscriptions even with the new API, if that makes the most sense for
their use case.

Coordinate with library authors to migrate to the new API. Once the major
libraries are ready, and we've allowed enough time for users to migrate, we will
deprecate the old API in a minor update.

In the following major release, we'll remove the old API.

# Unresolved questions

## Should we warn if a consumer is rendered without a matching provider?

There are valid use cases for relying on the default context value. In many
or most cases, it's likely a developer mistake. We could print a warning in
development when a consumer is rendered without a matching provider. To supress
the warning, the developer passes `true` to `allowDetached`.

```js
render() {
  return (
    // If there's no provider, this renders with the default theme.
    // `allowDetached` suppresses the development warning
    <ThemeContext.Consumer allowDetached={true}>
      {theme => (
        <h1 style={{color: theme === 'light' ? '#000' : '#fff'}}>
          {this.props.children}
        </h1>
      )}
    </ThemeContext.Consumer>
  );
}
```

## Add displayName argument to createContext for better debugging

For warnings and React DevTools, it would help if providers and consumers
had a `displayName`. The question is whether it should be required. We could
make it optional, and use a Babel transform to add the name automatically. This
is the strategy we use for `createClass`.

## Other

* Should the consumer use `children` as a render prop, or a named prop?
* How quickly should we deprecate and remove the existing context API?
* Need to run benchmarks to determine how fast this will be.
* Heuristics for caching.
* A high-priority version of this feature for animations. (May submit as a
  separate proposal.)
