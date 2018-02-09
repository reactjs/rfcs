- Start Date: (fill me in with today's date, 2018-01-25)
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Replace string refs with similar functionality to how they currently work but without many of the issues that string refs currently have (a list of them can be found at [#1373](https://github.com/facebook/react/issues/1373)).  

# Basic example

The React.createRef() API will create an immutable object ref (where its value is a mutable object referencing the actual ref). Accessing the ref value can be done via ref.value. An example of how this works is below:

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

The primary motivation is to encourage people to migrate off string refs. Callback refs meet some resistance because they are a bit harder to understand. The proposal introduces this API primarily for people who love string refs today.

There's a few problems with them.

Strings refs bind to the React's component's `currentOwner` rather than the parent. That's something that isn't statical analysable and leads to most of bugs.

```js
class ComponentA {
  render() {
    return (
      // ref foo1 would bind to ComponentA
      <div ref="foo1"/>
        <ComponentB>{
          // ref foo2 would bind to ComponentB
          // even though it's all in ComponentA's render
          () => <div ref="foo2" />
        }</ComponentB>
      </div>
    );
  }
}
```

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

Callback refs are easier to use when child with ref and parent have different lifetime. With object ref it can be achieved with `componentDidUpdate` hook. As present in the example below callback refs are less verbose in this case.

```js
class ComponentA extends React.Component {
  setRef = (element) => {
    if (element) {
      // element is HTMLElement
      // ref is created
    } else {
      // element is null
      // ref is destroyed
      // cached reference is required to stop listening events
    }
  }

  divRef = React.createRef();

  componentDidMount() {
    if (this.divRef.value) {
      // ref is created
    }
  }

  componentDidUpdate() {
    if (this.divRef.value) {
      // ref is created
    } else {
      // ref is null
      // should be used cached reference to stop listening events
    }
  }

  componentWillUnmount() {
    if (this.divRef.value) {
      // ref is HTMLElement, not removed yet
      // cached reference is not required
    }
  }

  render() {
    if (this.props.enabled) {
      return (
        <>
          <div ref={this.setRef} />
          <div ref={this.divRef} />
        </>
      );
    } else {
      return null;
    }
  }
}
```

And still in most cases the same parent and ref lifetime makes code cleaner and simpler.

# Alternatives

The common practice will be using ref objects (aka createRef), unless people have already been using callback refs and are happy with them.

Also there's no point in changing from callback refs to createRef right now unless people see value in doing so, otherwise it's just tech debt with no real additional value other than maybe some perf wins from not creating closures in the render method.

# Adoption strategy

Initially, we will release the object refs alongside the existing string refs as a minor update. Also this update will include `StrictMode` which enables deprecation messages for its subtree, so people be able to catch using string refs in their components and replace with the new api incrementally.

String refs will be removed in upcoming major release, so there will be enought time for migration.

# How we teach this

Callback and object refs solve different problems and crossing over in what they offer too - much like how components in React do currently.

Someone would likely choose an object ref when they only care about binding the object reference to the component instance/state they're in. Then the ref can be used for caching, or being passed to another component/handler in a lifecycle event.

Callback refs are great when someone likes to have find grain control over the node at the point where it gets provides and removed (like a subscription) to react to it.

# Unresolved questions

Optional, but suggested for first drafts. What parts of the design are still
TBD?
