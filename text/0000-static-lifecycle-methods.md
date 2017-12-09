- Start Date: 2017-12-08
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Replace error-prone render phase lifecycle hooks with static methods to make it easier to write async-compatible React components.

Provide a clear migration path for legacy components to become async-ready.

# Basic example

## Current API

The following example combines several patterns that I think are common in React components:

```js
class ExampleComponent extends React.Component {
  constructor(props) {
    super(props);

    this.state = {
      derivedData: null,
      externalData: null,
      someStatefulValue: null
    };
  }

  componentWillMount() {
    asyncLoadData(this.props.someId).then(externalData =>
      this.setState({ externalData })
    );

    // Note that this is not safe; (it can leak)
    // But it is a common pattern so I'm showing it here.
    addExternalEventListeners();

    this._computeMemoizedInstanceData(this.props);
  }

  componentWillReceiveProps(nextProps) {
    this.setState({
      derivedData: computeDerivedState(nextProps)
    });
  }

  componentWillUnmount() {
    removeExternalEventListeners();
  }

  componentWillUpdate(nextProps, nextState) {
    if (this.props.someId !== nextProps.someId) {
      asyncLoadData(nextProps.someId).then(externalData =>
        this.setState({ externalData })
      );
    }

    if (this.state.someStatefulValue !== nextState.someStatefulValue) {
      nextProps.onChange(nextState.someStatefulValue);
    }

    this._computeMemoizedInstanceData(nextProps);
  }

  render() {
    if (this.state.externalData === null) {
      return <div>Loading...</div>;
    }

    // Render real view...
  }
}
```

## Proposed API

This proposal would modify the above component as follows:

```js
class ExampleComponent extends React.Component {
  constructor(props) {
    super(props);

    this.state = {
      derivedData: null,
      externalData: null,
      someStatefulValue: null
    };
  }

  static deriveStateFromProps(props, state, prevProps) {
    // If derived state is expensive to calculate,
    // You can compare props to prevProps and conditionally update.
    return {
      derivedData: computeDerivedState(props)
    };
  }

  static prefetch(props, state) {
    // Prime the async cache early.
    // (Async request won't complete before render anyway.)
    // If you only need to pre-fetch before mount,
    // You can conditionally fetch based on state.
    asyncLoadData(props.someId);
  }

  componentDidMount() {
    // Event listeners are only safe to add after mount,
    // So they won't leak if mount is interrupted or errors.
    addExternalEventListeners();

    // Wait for earlier pre-fetch to complete and update state.
    // (This assumes some kind of cache to avoid duplicate requests.)
    asyncLoadData(props.someId).then(externalData =>
      this.setState({ externalData })
    );
  }

  componentDidUpdate(prevProps, prevState) {
    // Callbacks (side effects) are only safe after commit.
    if (this.state.someStatefulValue !== prevState.someStatefulValue) {
      this.state.onChange(this.state.someStatefulValue);
    }
  }

  componentWillUnmount() {
    removeExternalEventListeners();
  }

  render() {
    // Memoization that doesn't go in state can be done in render.
    // It should be idempotent and have no external side effects or mutations.
    // Examples include incrementing unique ids,
    // Lazily calculating and caching values, etc.
    this._computeMemoizedInstanceData();

    if (this.state.externalData === null) {
      return <div>Loading...</div>;
    }

    // Render real view...
  }
}

```

# Motivation

The React team recently added a feature flag to stress-test Facebook components for potential incompatibilities with our experimental async rendering mode ([facebook/react/pull/11587](https://github.com/facebook/react/pull/11587)). We enabled this feature flag internally so that we could:
1. Identify common problematic coding patterns with the legacy component API to inform a new async component API.
2. Find and fix async bugs before they impact end-users by intentionally triggering them in a deterministic way.
3. Gain confidence that our existing products could work in async.

I believe this GK confirmed what we suspected about the legacy component API: _It has too many potential pitfalls to be safely used for async rendering._

## Common problems

Some of the most common problematic patterns that were uncovered include:
* **Initializing Flux stores in `componentWillMount`**. It's often unclear whether this is an actual problem or just a potential one (eg if the store or its dependencies change in the future). Because of this uncertainty, it should be avoided.
* **Adding event listeners/subscriptions** in `componentWillMount` and removing them in `componentWillUnmount`. This causes leaks if the initial render is interrupted (or errors) before completion.
* **Non-idempotent external function calls** during `componentWillMount`, `componentWillUpdate`, or `componentWillReceiveProps` (eg registering callbacks that may be invoked multiple times, initializing or configuring shared controllers in such a way as to trigger invariants, etc.)

The [example above](#basic-example) attempts to illustrate a few of these patterns.

## Proposal

This proposal is intended to reduce the risk of writing async-compatible React components.

It does this by removing many<sup>1</sup> of the potential pitfalls in the current API while retaining important functionality the API enables. I believe this can be accomplished through a combination of:

1. Choosing lifecycle method names that have a clearer, more limited purpose.
2. Making certain lifecycles static to prevent unsafe access of instance properties.

<sup>1</sup> It is not possible to detect or prevent all side-effects (eg mutations of global/shared objects).

# Detailed design

## New static lifecycle methods

### `static prefetch(props: Props, state: State): void`

This method is invoked before `render` for both the initial render and all subsequent updates. It is not called during server rendering.

The purpose of this method is to initiate asynchronous request(s) as early as possible in a component's rendering lifecycle. Such requests will not block `render`. They can be used to pre-prime a cache that is later used in `componentDidMount`/`componentDidUpdate` to trigger a state update.

Avoid introducing any side-effects, mutations, or subscriptions in this method. For those use cases, use `componentDidMount`/`componentDidUpdate` instead.

### `static deriveStateFromProps(props: Props, state: State, prevProps: Props): PartialState | null`

This method is invoked before a mounted component receives new props. Return an object to update state in response to prop changes.

Note that React may call this method even if the props have not changed. I calculating derived data is expensive, compare new and previous prop values to conditionally handle changes.

React does not call this method before the intial render/mount and so it is not called during server rendering.

## Deprecated lifecycle methods

### `componentWillMount` -> `unsafe_componentWillMount`

This method will log a deprecation warning in development mode recommending that users either rename to `unsafe_componentWillMount` or use the new static `prefetch` method instead. It will be removed entirely in version 17.

### `componentWillUpdate` -> `unsafe_componentWillUpdate`

This method will log a deprecation warning in development mode recommending that users either rename to `unsafe_componentWillUpdate` or use the new static `prefetch` method instead. It will be removed entirely in version 17.

### `componentWillReceiveProps` -> `unsafe_componentWillReceiveProps`

This method will log a deprecation warning in development mode recommending that users either rename to `unsafe_componentWillReceiveProps` or use the new static `deriveStateFromProps` method instead. It will be removed entirely in version 17.

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