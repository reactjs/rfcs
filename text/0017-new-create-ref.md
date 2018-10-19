- Start Date: 2018-01-25
- RFC PR: https://github.com/reactjs/rfcs/pull/17
- React Issue: https://github.com/facebook/react/issues/10581

# Summary

Currently there are two ref APIs in React: string refs and callback refs.

String refs are considered legacy due to [numerous issues](https://github.com/facebook/react/issues/1373) in their design. [Callback refs](https://reactjs.org/docs/refs-and-the-dom.html) don't share these deficiencies and were introduced to replace string refs. However, there is more ceremony around writing them, they have some [unintuitive caveats](https://reactjs.org/docs/refs-and-the-dom.html#caveats-with-callback-refs) and can be hard to teach.

**The goal of this RFC is to introduce a new intuitive ref API. It is very similar in its "feel" to string refs, but doesn't suffer from their problems.** It is intentionally less poweful than the callback ref API. The plan is to:

* Keep the callback ref API for more advanced use cases.
* Replace the string refs by the newly proposed "object" ref API.

String refs would get deprecated in a minor release, and support for them would eventually get removed in an upcoming major release.

# Basic example

The `React.createRef()` API will create an immutable object ref (where its value is a mutable object referencing the actual ref). Accessing the ref value can be done via `ref.current`. An example of how this works is below:

```js
class MyComponent extends React.Component {
  divRef = React.createRef();

  render() {
    return <div><input type="text" ref={this.divRef} /></div>;
  }

  componentDidMount() {
    this.divRef.current.focus();
  }
}
```

# Motivation

The primary motivation is to encourage people to migrate off string refs. Callback refs meet some resistance because they are a bit harder to understand. The proposal introduces this API primarily for people who love string refs today.

### What's wrong with string refs?

The main problem with string refs is that they require React to keep track of the "currently executing" component. However, that component doesn't always match where the string ref is defined. Consider the common "render prop" pattern:

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

This behavior is highly confusing.

There are other issues with string refs in practice:

* They cause bugs when [multiple React copies are on the same page](https://reactjs.org/warnings/refs-must-have-owner.html).
* They're not friendly to static typing.
* They aren't compatible with advanced compilation strategies that mangle class properties.
* They can't be [composed](https://github.com/ide/react-clone-referenced-element/).

### Don't we have callback refs for this?

Many people continue to favor string refs because writing callback refs requires more mental overhead (you have to think about a field *and* a function that sets it). They are especially inconvenient when you need an array of refs. Callback refs also have [unusual caveats](https://reactjs.org/docs/refs-and-the-dom.html#caveats-with-callback-refs) that are [often](https://github.com/facebook/react/issues/9328) [mistaken for a bug](https://github.com/facebook/react/issues/8619).

Callback refs are, strictly saying, more powerful than either string refs or the proposed object refs. However, there is definitely a niche for a simpler, more convenient API for the majority of cases. There might be some small wins in performance too: commonly, ref value is assigned in a closure created in the render phase of a component. This API avoids that. Similarly, this avoids assigning to a non-existent component instance property, which also can cause deopts in JavaScript engines.

# Detailed design

Introduces `React.createRef()`:

```js
type RefObject<T> = {
  current: T | null
};

interface React {
  createRef<T>(): RefObject<T>;
}
```

`createRef` requires a explicit type annotation to ensure type correctness.

Before `componentDidMount` lifecycle `divRef.current` will be filled with element/component reference:

```js
componentDidMount() {
  this.divRef.current instanceof HTMLDivElement === true
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

Callback refs are easier to use when child with ref and parent have different lifetime. Consider this example:

```js
class CallbackRefExample extends React.Component {
  // Callback refs offer a centralized place
  // to perform side effects.
  divRef = (div) => {
    if (div) {
      this.div = div;
      this.div.addEventListener('foo', onFoo);
    } else {
      this.div.removeEventListener('foo', onFoo);
      this.div = null;
    }
  }

  onFoo = () => {
    console.log('hello');
  }

  render() {
    if (this.props.enabled) {
      return <div ref={this.divRef} />;
    } else {
      return null;
    }
  }
}
```

We could write an equivalent example with object refs by using `componentDidMount` and `componentDidUpdate`. However, by the time `componentDidUpdate` fires, we have no way to access the previous ref value to remove the listener. So we'd have to "mirror" the ref value in a separate instance variable. This becomes very tedious and error-prone.

The conclusion here is that for advanced use cases (such as side effects during ref attachment and detachment), callback refs are preferable. Still, in most cases this is not necessary, and object refs are sufficient.

# Alternatives

One potential alternative is to do nothing, and keep string refs and callback refs. The problem with keeping string refs is that they prevent some future work on React, including advanced compilation strategies. So we need a migration path off of them.

Another potential alternative is to fully embrace callback refs but deprecate string refs. However, there is a lot of feedback suggesting that people struggle with understanding and using the callback ref API, and a simpler option is necessary.

The `createRef` API could potentially be implemented as a userland package. However, it would be a one-liner, and most people would not know about it. If we wanted everybody to use it, we might as well build it into React.

# Adoption strategy

Initially, we will release the object refs alongside the existing string and callback refs as a minor update. This update will also include a `<StrictMode>` component which enables deprecation messages for its subtree. This lets people catch string refs usage in their components and replace with the new API (or callback refs, if they prefer) incrementally.

String refs will be removed in an upcoming major release, so there will be enough time for migration.

# How we teach this

Callback and object refs solve different problems and crossing over in what they offer too - much like how components in React do currently.

In most cases people would choose an object ref. They could use it when they only care about accessing the object reference. The ref object can be passed around, or stored in an array.

Callback refs will be positioned as an advanced API. It is useful when someone likes to have fine-grained control over the node when it gets attached and detached, or to compose multiple refs.

# Unresolved questions

None.
