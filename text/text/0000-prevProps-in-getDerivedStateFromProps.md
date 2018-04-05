- Start Date: (2018-04-05)
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Not having `prevProps` in the new `getDerivedStateFromProps()` function is really inconvenient for a library I'm migrating because in `componentWillReceiveProps()` it compared a property to find out if it has changed, and only if it did then the component changed its own internal state.

# Basic example

I added `prevProps` to `getDerivedStateFromProps()` myself this way:

```js
constructor(props) {
  super(props)

  // Needs to store initial `this.props` here
  // so that it's not `undefined` on the first
  // `getDerivedStateFromProps()` call.
  this.state = { props }
}

// `state.props` is `prevProps`.
static getDerivedStateFromProps(props, state) {
  if (props.country !== state.props.country) {
    return {
      props,
      derivedValue: ...
    }
  }
}
```

The whole `constructor()` thing is just for `prevProps` and feels bulky.

And I could be comparing more than just country - I'm also comparing `localeMessages` so that if they did change then I do

```js
// Imagine a user switched their locale, so the UI adapts in real-time.
if (prevProps.localeMessages !== props.localeMessages) {
  this.setState({
    selectOptions: generateLocalizedSelectOptions(props.localeMessages)
  })
}
```

# Motivation

In my library `componentWillReceiveProps()` compared a property to find out if it has changed, and if it did then the component changed its own internal state. There's a suggestion to use `componentDidUpdate()` but it feels smelly because "derive state from props" is what my old `componentWillReceiveProps` really does so the name kinda fits in only if it did have `prevProps` argument.

# Detailed design

`getDerivedStateFromProps()` could take a third `prevProps` argument, with the first call simply being `getDerivedStateFromProps(this.props, this.state, this.props)` (inside constructor(), or whatever else it's now at). The first call would simply be a no-op.

This behaviour complies with the official docs [explicitly state](https://reactjs.org/docs/react-component.html#static-getderivedstatefromprops):

> Note that if a parent component causes your component to re-render, this method will be called even if props have not changed. You may want to compare new and previous values if you only want to handle changes.

I.e. it says that the new props aren't neccessarily different from the old ones, so doing `getDerivedStateFromProps(this.props, this.state, this.props)` wouldn't be illegal in any way and wouldn't contradict the already cemented behaviour for this new function.

# Drawbacks

None I can think of.

# Alternatives

I guess this would work but it doesn't feel semantically right given that there already is `getDerivedStateFromProps()`.

```js
componentDidUpdate(prevProps) {
  if (this.props.country !== prevProps.country) {
    this.setState({
      derivedValue: ...
    })
  }
}
```

# Adoption strategy

If the third argument is added there's no migration strategy. It could be the first one, but React 16.3 is already released.
