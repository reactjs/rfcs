- Start Date: 2017-12-08
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Replace error-prone render phase lifecycle hooks with static methods to make it easier to write async-compatible React components.

Provide a clear migration path for legacy components to become async-ready.

# Basic example

At a high-level, I propose the following additions/changes to the component API. (The motivations for these changes are explained below.)

```js
class ExampleComponent extends React.Component {
  static getDerivedStateFromNextProps(nextProps, prevProps, prevState) {
    // Called before a mounted component receives new props.
    // Return an object to update state in response to prop changes.
    // Return null to indicate no change to state.
  }

  static optimisticallyPrepareToRender(props, state) {
    // Initiate async request(s) as early as possible in rendering lifecycle.
    // These requests do not block `render`.
    // They can only pre-prime a cache that is used later to update state.
    // (This is a micro-optimization and probably not a common use-case.)
  }

  unsafe_componentWillMount() {
    // New name for componentWillMount()
    // Indicates that this method can be unsafe for async rendering.
  }

  unsafe_componentWillUpdate(nextProps, nextState) {
    // New name for componentWillUpdate()
    // Indicates that this method can be unsafe for async rendering.
  }

  unsafe_componentWillReceiveProps(nextProps) {
    // New name for componentWillReceiveProps()
    // Indicates that this method can be unsafe for async rendering.
  }
}
```

# Motivation

The React team recently added a feature flag to stress-test Facebook components for potential incompatibilities with our experimental async rendering mode ([facebook/react/pull/11587](https://github.com/facebook/react/pull/11587)). We enabled this feature flag internally so that we could:
1. Identify common problematic coding patterns with the legacy component API to inform a new async component API.
2. Find and fix async bugs before they impact end-users by intentionally triggering them in a deterministic way.
3. Gain confidence that our existing products could work in async.

I believe this internal experiment confirmed what we suspected about the legacy component API: _It has too many potential pitfalls to be safely used for async rendering._

## Common problems

Some of the most common problematic patterns that were uncovered include:
* **Initializing Flux stores in `componentWillMount`**. It's often unclear whether this is an actual problem or just a potential one (eg if the store or its dependencies change in the future). Because of this uncertainty, it should be avoided.
* **Adding event listeners/subscriptions** in `componentWillMount` and removing them in `componentWillUnmount`. This causes leaks if the initial render is interrupted (or errors) before completion.
* **Non-idempotent external function calls** during `componentWillMount`, `componentWillUpdate`, `componentWillReceiveProps`, or `render` (eg registering callbacks that may be invoked multiple times, initializing or configuring shared controllers in such a way as to trigger invariants, etc.)

## Goal

The goal of this proposal is to reduce the risk of writing async-compatible React components. I believe that can be accomplished by removing many<sup>1</sup> of the potential pitfalls in the current API while retaining important functionality the API enables. This can be done through a combination of:

1. Choosing lifecycle method names that have a clearer, more limited purpose.
2. Making certain lifecycles static to prevent unsafe access of instance properties.

<sup>1</sup> It is not possible to detect or prevent all side-effects (eg mutations of global/shared objects).

## Examples

Let's look at some of the common usage patterns mentioned above and how they might be adapted to the new proposed API.

### Prefetching async data during mount

The purpose of this pattern is to initiate data loading as early as possible.

It is worth noting that in both examples below, the data will not finish loading before the initial render (so a second render pass will be required in either case).

#### Before

```js
class ExampleComponent extends React.Component {
  state = {
    externalData: null,
  };

  componentWillMount() {
    asyncLoadData(this.props.someId).then(externalData =>
      this.setState({ externalData })
    );
  }

  render() {
    if (this.state.externalData === null) {
      // Render loading UI...
    } else {
      // Render real view...
    }
  }
}
```

#### After

```js
class ExampleComponent extends React.Component {
  state = {
    externalData: null,
  };

  static optimisticallyPrepareToRender(props, state) {
    // Prime an external cache as early as possible.
    // (Async request won't complete before render anyway.)
    if (state.externalData === null) {
      asyncLoadData(props.someId);
    }
  }

  componentDidMount() {
    // Wait for earlier pre-fetch to complete and update state.
    // (This assumes some kind of cache to avoid duplicate requests.)
    asyncLoadData(props.someId).then(externalData => {
      // Note that if the component unmounts before this request completes,
      // It will trigger a warning, "cannot update an unmounted component".
      // You can avoid this by tracking mounted state with an instance var if desired.
      this.setState({ externalData });
    });
  }

  render() {
    if (this.state.externalData === null) {
      // Render loading UI...
    } else {
      // Render real view...
    }
  }
}
```

### State derived from props/state

The purpose of this pattern is to calculate some values derived from props for use during render.

Typically `componentWillReceiveProps` is used for this, although if the calculation is fast enough it could just be done in `render`.

#### Before

```js
class ExampleComponent extends React.Component {
  state = {
    derivedData: computeDerivedState(this.props)
  };

  componentWillReceiveProps(nextProps) {
    if (this.props.someValue !== nextProps.someValue) {
      this.setState({
        derivedData: computeDerivedState(nextProps)
      });
    }
  }
}
```

#### After

```js
class ExampleComponent extends React.Component {
  state = {
    derivedData: computeDerivedState(this.props)
  };

  static getDerivedStateFromNextProps(nextProps, prevProps, prevState) {
    if (nextProps.someValue !== prevProps.someValue) {
      return {
        derivedData: computeDerivedState(nextProps)
      };
    }

    // Return null to indicate no change to state.
    return null;
  }
}
```

### Adding event listeners/subscriptions

The purpose of this pattern is to subscribe a component to external events when it mounts and unsubscribe it when it unmounts.

The `componentWillMount` lifecycle is often used for this purpose, but this is problematic because any interruption _or_ error during initial mount will cause a memory leak. (The `componentWillUnmount` lifecycle hook is not invoked for a component that does not finish mounting and so there's no safe place to handle unsubscriptions in that case.)

Using `componentWillMount` for this purpose might also cause problems in the context of server-rendering.

#### Before

```js
class ExampleComponent extends React.Component {
  componentWillMount() {
    this.setState({
      subscribedValue: this.props.dataSource.value
    });

    // This is not safe; (it can leak).
    this.props.dataSource.subscribe(this._onSubscriptionChange);
  }

  componentWillUnmount() {
    this.props.dataSource.unsubscribe(this._onSubscriptionChange);
  }

  render() {
    // Render view using subscribed value...
  }

  _onSubscriptionChange = subscribedValue => {
    this.setState({ subscribedValue });
  };
}
```

#### After

```js
class ExampleComponent extends React.Component {
  state = {
    subscribedValue: this.props.dataSource.value
  };

  componentDidMount() {
    // Event listeners are only safe to add after mount,
    // So they won't leak if mount is interrupted or errors.
    this.props.dataSource.subscribe(this._onSubscriptionChange);

    // External values could change between render and mount,
    // In some cases it may be important to handle this case.
    if (this.state.subscribedValue !== this.props.dataSource.value) {
      this.setState({
        subscribedValue: this.props.dataSource.value
      });
    }
  }

  componentWillUnmount() {
    this.props.dataSource.unsubscribe(this._onSubscriptionChange);
  }

  render() {
    // Render view using subscribed value...
  }

  _onSubscriptionChange = subscribedValue => {
    this.setState({ subscribedValue });
  };
}
```

### External function calls (side effects, mutations)

The purpose of this pattern is to send an external signal that something has changed internally (eg in `state`).

The `componentWillUpdate` lifecycle hook is sometimes used for this but it is not ideal because this method may be called multiple times _or_ called for props that are never committed. `componentDidUpdate` should be used for this purpose rather than `componentWillUpdate`.

#### Before

```js
class ExampleComponent extends React.Component {
  componentWillUpdate(nextProps, nextState) {
    if (this.state.someStatefulValue !== nextState.someStatefulValue) {
      nextProps.onChange(nextState.someStatefulValue);
    }
  }
}
```

#### After

```js
class ExampleComponent extends React.Component {
  componentDidUpdate(prevProps, prevState) {
    // Callbacks (side effects) are only safe after commit.
    if (this.state.someStatefulValue !== prevState.someStatefulValue) {
      this.props.onChange(this.state.someStatefulValue);
    }
  }
}
```

### Memoized values derived from `props` and/or `state`

The purpose of this pattern is to memoize computed values based on `props` and/or `state`.

Typically such values are stored in `state`, but in some cases the values require mutation and as such may not seem suited for state (although they could technically still be stored there). An example of this would be an external helper class that calculates and memoizes values interally.

In other cases the value may be derived from `props` _and_ `state`.

#### Before

```js
class ExampleComponent extends React.Component {
  componentWillMount() {
    this._calculateMemoizedValues(this.props, this.state);
  }

  componentWillUpdate(nextProps, nextState) {
    if (
      this.props.someValue !== nextProps.someValue ||
      this.state.someOtherValue !== nextState.someOtherValue
    ) {
      this._calculateMemoizedValues(nextProps, nextState);
    }
  }

  render() {
    // Render view using calculated memoized values...
  }
}
```

#### After

```js
class ExampleComponent extends React.Component {
  render() {
    // Memoization that doesn't go in state can be done in render.
    // It should be idempotent and have no external side effects or mutations.
    this._calculateMemoizedValues(this.props, this.state);

    // Render view using calculated memoized values...
  }
}
```

### Initializing Flux stores during mount

The purpose of this pattern is to initialize some Flux state when a component is mounted.

This is sometimes done in `componentWillMount` which can be problematic if, for example, the action is not idempotent. From the point of view of the component, it's often unclear whether dispatching the action more than once will cause a problem. It's also possible that it does not cause a problem when the component is authored but later does due to changes in the store. Because of this uncertainty, it should be avoided.

We recommend using `componentDidMount` for such actions since it will only be invoked once.

#### Before

```js
class ExampleComponent extends React.Component {
  componentWillMount() {
    FluxStore.dispatchSomeAction();
  }
}

```

#### After

```js
class ExampleComponent extends React.Component {
  componentDidMount() {
    // Side effects (like Flux actions) should only be done after mount or update.
    // This prevents duplicate actions or certain types of infinite loops.
    FluxStore.dispatchSomeAction();
  }
}
```

# Detailed design

## New static lifecycle methods

### `static getDerivedStateFromNextProps(nextProps: Props, prevProps: Props, prevState: Props): PartialState | null`

This method is invoked before a mounted component receives new props. Return an object to update state in response to prop changes. Return null to indicate no change to state.

Note that React may call this method even if the props have not changed. If calculating derived data is expensive, compare next and previous props to conditionally handle changes.

React does not call this method before the intial render/mount and so it is not called during server rendering.

### `static optimisticallyPrepareToRender(props: Props, state: State): void`

This method is invoked before `render` for both the initial render and all subsequent updates. It is not called during server rendering.

The purpose of this method is to initiate asynchronous request(s) as early as possible in a component's rendering lifecycle. Such requests will not block `render`. They can be used to pre-prime a cache that is later used in `componentDidMount`/`componentDidUpdate` to trigger a state update.

Avoid introducing any non-idempotent side-effects, mutations, or subscriptions in this method. For those use cases, use `componentDidMount`/`componentDidUpdate` instead.

## Deprecated lifecycle methods

### `componentWillMount` -> `unsafe_componentWillMount`

This method will log a deprecation warning in development mode recommending that users either rename to `unsafe_componentWillMount` or use the new static `optimisticallyPrepareToRender` method instead. It will be removed entirely in version 17.

### `componentWillUpdate` -> `unsafe_componentWillUpdate`

This method will log a deprecation warning in development mode recommending that users either rename to `unsafe_componentWillUpdate` or use the new static `optimisticallyPrepareToRender` method instead. It will be removed entirely in version 17.

### `componentWillReceiveProps` -> `unsafe_componentWillReceiveProps`

This method will log a deprecation warning in development mode recommending that users either rename to `unsafe_componentWillReceiveProps` or use the new static `getDerivedStateFromNextProps` method instead. It will be removed entirely in version 17.

# Drawbacks

The current component lifecycle hooks are familiar and used widely. This proposed change will introduce a lot of churn within the ecosystem. I hope that we can reduce the impact of this change through the use of codemods, but it will still require a manual review process and testing.

This change is **not fully backwards compatible**. Libraries will need to drop support for older versions of React in order to use the new, static API. Unfortunately, I believe this is unavoidable in order to safely transition to an async-compatible world.

# Alternatives

## Try to detect problems using static analysis

It is possible to create ESLint rules that attempt to detect and warn about potentially unsafe actions inside of render-phase lifecycle hooks. Such rules would need to be very strict though and would likely result in many false positives. It would also be difficult to ensure that library maintainers correctly used these lint rules, making it possible for async-unsafe components to cause problems within an async tree.

Sebastian has also discussed the idea of side effect tracking with the Flow team. Conceptually this would enable us to know, statically, whether a method is free of side effects and mutations. This functionality does not currently exist in Flow though, and if it did there will still be an adoption problem. (Not everyone uses Flow and there's no way to gaurantee the shared components you rely on are safe.)

## Don't support async with the legacy class component API

We could leave the class component API as-is and instead focus our efforts on a new stateful, functional component API. If a legacy class component is detected within an async tree, we could revert to sync rendering mode.

There are no advanced proposals for such a stateful, functional component API that I'm aware of however, and the complexity of such a migration would likely be at least as large as this proposal.

# Adoption strategy

Begin by reaching out to prominent third-party library maintainers to make sure there are no use-cases we have failed to consider.

Assuming we move forward with the proposal, release (at least one) minor 16.x update to add deprecation warnings for the legacy lifecycles and inform users to either rename with the `unsafe_` prefix or use the new static methods instead. We'll then cordinate with library authors to ensure they have enough time to migrate to the new API in advance of the major release that drops support for the legacy lifecycles.

We will provide a codemod to rename the deprecated lifecycle hooks with the new `unsafe_` prefix.

We will also provide codemods to assist with the migration to static methods, although given the nature of the change, codemods will be insufficient to handle all cases. Manual verification will be required.

# How we teach this

Write a blog post (or a series of posts) announcing the new lifecycle hooks and explaining our motivations for the change, as well as the benefits of being async-compatible. Provide examples of how to migrate the most common legacy patterns to the new API. (This can be more detailed versions of the [basic example](#basic-example) shown in the beginning of this RFC.)

# Unresolved questions

## Can `shouldComponentUpdate` remain an instance method?

Anectdotally, it seems far less common for this lifecycle hook to be used in ways that are unsafe for async. The overwhelming common usagee of it seems to be returning a boolean value based on the comparison of current to next props.

On the one hand, this means the method could be easily codemodded to a static method, but it would be equally easy to write a custom ESLint rule to warn about `this` references to anything other than `this.props` inside of `shouldComponentUpdate`.

Beyond this, there is some concern that making this method static may complicate inheritance for certain languages/compilers.

## Can `render` remain an instance method?

There primary motivation for leaving `render` as an instance method is to allow other instance methods to be used as event handlers and ref callbacks. (It is important for event handlers to be able to call `this.setState`.) We may change the event handling API in the future to be compatible with eg error boundaries, at which point it might be appropriate to revisit this decision.

Leaving `render` as an instance method also provides a mechanism (other than `state`) on which to store memoized data.

## Other

Are there important use cases that I've overlooked that the new static lifecycles would be insufficient to handle?
