- Start Date: 2018-05-9
- RFC PR:
- React Issue:

# Summary

Give the developer a chance to `intentially` patch up the mismatches during hydration.

# Basic example

```jsx

constructor(props) {
  super(props)

  this.styles = Object.defineProperties({}, {
    'container': { get: _=> {
       return typeof window == 'undefined'
       ? { height: 300 }
       : { height: this.props.viewsize.height * 0.67}
    }}
  })
}

render() {
  let s = this.styles
  return <div ssrReconciliation style={s['container']} />
}


```

# Motivation

React expects that the rendered content is identical between the server and the client. This is important for performance reasons. If you intentionally need to render something different on the server and the client, you have to do a two-pass rendering like [facebook/react/issues/8017#issuecomment-256351955](https://github.com/facebook/react/issues/8017#issuecomment-256351955)


If an attribute is derived from screen size, since we don't know the real screen size in the server side, we might do a 2 pass rendering like following:

```jsx
constructor(props) {
  super(props)
  this.state = { hasMounted: false }

  this.styles = Object.defineProperties({}, {
    'container': { get: _=> {
       if (typeof window == 'undefined') return { height: 300 }
       let height = this.props.viewsize.height * 0.67
       return this.state.hasMounted ? { height } : { height: height - 1 }
    }}
  })
}

componentDidMount() {
    this.setState({ hasMounted: true })
}

render() {
  let s = this.styles
  return (
    <div  style={s['container']}>
  </div>
  )
}
```

In the above code, I demonstrate that I have to intentionally render a wrong height initially in the browser so I can update it in the second run, or the DOM will keep inconsistent with the vDOM.

This is because if the 2 renderings keep unchanged in vDOM, React won't update the DOM. Two pass rendering to patch up the mismatches comes with some drawbacks:

1. Introduce unnecessary 2nd rendering

2. The component will do 2 pass rendering in every mounting even it is not in hydration.

3. Limited to stateful component.

# Detailed design

Design a `ssrReconciliation={true}` attribute, or something like that, e.g. ssrPatchUp={true} for the developer to ask React to patch up if a single element's attribute or text content is unavoidably different between the server and the client during hydration.


# Drawbacks

Increased validating cost during hydration, and the developer might overuse this to prevent any mismatch in the future.

# Alternatives

Unsure

# Adoption strategy

This is not a breaking change.

# How we teach this

Update hydrate() to replace two-pass rendering by this new attribute.
