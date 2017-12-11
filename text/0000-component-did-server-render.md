- Start Date: 2017-12-11
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Provide a server-side rendering equivalent for `componentDidMount`.

This RFC relates to the proposal to deprecate `componentWillMount` ([RFC #6](https://github.com/reactjs/rfcs/pull/6)) and has been [previously discussed on GitHub](https://github.com/facebook/react/issues/7671).

# Basic example

Because `componentWillMount` is the only lifecycle method currently invoked during server rendering, it is sometimes used for things like logging, eg:
```js
class Example extends React.Component {
  componentWillMount() {
    if (typeof window === 'undefined') {
      // Server-side
    }
  }

  componentDidMount() {
    // Client-side
  }
}
```

This RFC proposes to add a new lifecycle hook named `componentDidServerRender` to be called after initial server render, eg:
```js
class Example extends React.Component {
  componentDidServerRender() {
    // Server-side
  }

  componentDidMount() {
    // Client-side
  }
}
```

# Motivation

Provide a place for initialization logic with side effects to safely run on the server.

Typically, `componentWillMount` is used for this, but [RFC #6](https://github.com/reactjs/rfcs/pull/6) proposes to remove this method. (It is not a very ergonomic solution anyway.) The logic could be moved to the component constructor but it's generally considered bad practice for a constructor to have side effects.

# Detailed design

## `componentDidServerRender()`

`componentDidServerRender()` is invoked immediately after a component is server rendered. It is the server equivalent to `componentDidMount()`.

Calling `setState()` in this method will trigger an extra rendering, but it is guaranteed to flush during the same tick. This guarantees that even though the `render()` will be called twice in this case, the user won't see the intermediate state.

Use this pattern with caution because it can impact performance. However, it is sometimes necessary for cases like localization or routing where you need to override the results of the initial render based on new information.

# Drawbacks

This proposal is backwards compatible.

# Alternatives

The alternative to this new lifecycle hook is to continue to feature test (eg `typeof window`) inside of either `componentWillMount` or the class constructor.

# Adoption strategy

Release a minor update to version 16 with support for the new lifecycle hook.

Pending the outcome of [RFC #6](https://github.com/reactjs/rfcs/pull/6), include messaging about this new hook in the `componentWillMount` deprecation warning.

Coordinate with popular libraries (eg [react-router](https://reacttraining.com/react-router/), [react-intl](https://github.com/yahoo/react-intl)) to ensure this new hook meets their needs before `componentWillMount` is deprecated. 

# How we teach this

Write a blog post for [reactjs.org](https://reactjs.org/) explaining the motivation for this new hook as well as showing a basic example of its usage. Add it to the [component reference API](https://reactjs.org/docs/react-component.html) as well.

# Unresolved questions

None?