- Start Date: (fill me in with today's date, 2018-01-25)
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Replace string refs with similar functionality to how they currently work but without many of the issues that string refs currently have (a list of them can be found at [#1373](https://github.com/facebook/react/issues/1373)).  

# Basic example

The React.createRef() API will create an immutable object ref (where it's value is a mutable object referencing the actual ref). Accessing the ref value can be done via ref.value. An example of how this works is below:

```js
class MyComponent extends React.Component {
  divRef = React.createRef();

  render() {
    return <div><input type="text" ref={this.divRef} /></div>;
  }

  componentDidMount() {
    this.divRef.value.focus();
  }
}
```

# Motivation

This alternative API shouldn't provide any big real wins over callback refs - other than being a nice convenience feature. There might be some small wins in performance - as a common pattern is to assign a ref value in a closure created in the render phase of a component - this avoids that (even more so when a ref property is assigned to a non-existent component instance property).

One benefit would be more correct Flow types. People tend to type refs incorrectly because Flow doesn't enforce uninitialized class properties correctly:

```js
class Foo { neverSet: boolean; }
let foo = new Foo();
(foo.neverSet: boolean); // works
```

# Detailed design

Introduces `React.createRef()`:

```js
type RefObject<T> = {
  value: T | null
};

interface React {
  createRef<T>(): RefObject<T>;
}
```

`createRef` requires a explicit type annotation to ensure type correctness.

After `componentDidMount` lifecycle `divRef.value` will be filled with element/component reference:

```js
componentDidMount() {
  this.divRef.value instanceof HTMLDivElement === true
}

render() {
  return (
    <div ref={this.divRef}>
      {this.props.children}
    </div>
  );
}
```

# Drawbacks

Why should we *not* do this? Please consider:

- implementation cost, both in term of code size and complexity
- whether the proposed feature can be implemented in user space
- the impact on teaching people React
- integration of this feature with other existing and planned features
- cost of migrating existing React applications (is it a breaking change?)

There are tradeoffs to choosing any path. Attempt to identify them here.

# Alternatives

What other designs have been considered? What is the impact of not doing this?

# Adoption strategy

If we implement this proposal, how will existing React developers adopt it? Is
this a breaking change? Can we write a codemod? Should we coordinate with
other projects or libraries?

# How we teach this

Callback and object refs solve different problems and crossing over in what they offer too - much like how components in React do currently.

Someone would likely choose an object ref when they only care about binding the object reference to the component instance/state they're in. Then the ref can be used for caching, or being passed to another component/handler in a lifecycle event.

Callback refs are great when someone likes to have find grain control over the node at the point where it gets provides and removed (like a subscription) to react to it.

# Unresolved questions

Optional, but suggested for first drafts. What parts of the design are still
TBD?
