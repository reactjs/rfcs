- Start Date: 2019-02-20
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

NOTE: This proposal will sound scary. Keep in mind, while reading this, that the actual upgrade path should be very simple for most people since the deprecated things are mostly edge cases and any common ones can be codemodded.

This proposal simplifies how React.createElement works and ultimately lets us remove the need for forwardRef.

- Deprecate "module pattern" components.
- Deprecate defaultProps on function components.
- Deprecate spreading `key` from objects.
- Deprecate string refs (and remove production mode `_owner` field).
- Move `ref` extraction to class render time and `forwardRef` render time.
- Move `defaultProps` resolution to class render time.
- Change JSX transpilers to use a new element creation method.
  - Always pass children as props.
  - Pass `key` separately from other props.
  - In DEV,
    - Pass a flag determining if it was static or not.
    - Pass `__source` and `__self` separately from other props.

The goal is to bring element creation down to this logic:

```
function jsx(type, props, key) {
  return {
    $$typeof: ReactElementSymbol,
    type,
    key,
    props,
  };
}
```

# Motivation

In React 0.12 time frame we did a bunch of small changes to how `key`, `ref` and `defaultProps` works. Particularly, they get resolved early on in the `React.createElement(...)` call. This made sense when everything was classes, but since then, we've introduced function components. Hooks have also make function components more prevalent. It might be time to reevaluate some of those designs to simplify things (at least for function components).

Element creation is a hot path because it is used a lot but also because it is always recreated in rerenders.

`React.createElement(...)` was never intended to be the implementation of JSX but was the best we could do with tooling at the time. It was intended as what you might write manually (if you didn't want to use the createFactory form). The alternatives never provided enough value to warrant rolling them out everywhere. It has a number of issues:

- We need to do a dynamic test against a component if it has a `.defaultProps` during every element creation call. This can't be optimized well because the function it is called within is highly megamorphic.
- `.defaultProps` in element creation doesn't work with `React.lazy` so in that case we also have to check for resolving defaultProps in the render phase too, and means that the semantics are inconsistent anyway.
- Children are passed as var args and we have to patch them onto props dynamically instead of statically knowning the shape of the props at the callsite.
- The transform uses `React.createElement` which is a dynamic property look up instead of a constant closed over module scope. This minimizes poorly and takes a little cost to run.
- We don't know if the passed-in props is a user-created object that can be mutated so we must always clone it once.
- `key` and `ref` gets extracted from JSX props provided so even if we didn't clone, we'd have to delete a prop, which would cause that object to become map-like.
- `key` and `ref` can be spread in dynamically so without prohibitive analysis, we don't know if these patterns will include them `<div {...props} />`.
- The transform relies on a the name `React` being in scope of JSX. I.e. you have to import the default. This is unfortunate as more things like Hooks are typically used as named arguments. Ideally, you wouldn't have to import anything to use JSX.

Other than performance this is also about a long term simplification of number of concepts you have to learn to use React. In particular, `forwardRef` and `defaultProps` are no longer something special.

Additionally, if we ever want to standardize more of JSX we need to start moving away from some of the more esoteric legacy behaviors of React.

# Detailed design

The design will include three steps. 1) A new JSX transform. 2) Deprecations and warnings. 3) Actual semantic breakages.

## JSX transform changes

There are many combinations of transpilers, bundlers, and downstream tooling that need to be updated to make any changes to React JSX support.

### Auto-import

The first thing we need to change is getting rid of the requirement to have a `React` identifier in scope.

Ideally, the element creation should be part of the transpiler's own runtime. There are some practical concerns. For one, we have both DEV mode and PROD mode. The DEV mode version is a lot more complicated and integrated into React. We also make subtle changes between versions - such as this one.

It's much easier to iterate on new versions by deploying npm packages than updates to the compiler toolchain. Therefore, it might be best if the actual implementation still lives in the `react` package.

Ideally, you wouldn't need to write any imports to use JSX:

```js
function Foo() {
  return <div />;
}
```

Then it would compile to include this dependency, and the subsequently, the bundler would resolve it to whatever it wants.

```js
import {jsx} from "react";
function Foo() {
  return jsx('div', ...);
}
```

The problem is that not all tooling supports adding new dependencies from a transform. The first step is figuring out how this can be done idiomatically in the current ecosystem.

### Pass `key` separately from props

Currently, key is passed as part of props but we'll want to special case it in the future so we need to pass it as a separate argument.

```js
jsx('div', props, key)
```

### Always pass children as props

In `createElement`, children gets passed as var args. In the new transform, we'll always just add them to the props object - inline.

The reason we pass them as var args is to distinguish static children from dynamic ones in DEV. We can instead pass a boolean or use two different functions to distinguish them. My proposal is that we compile `<div>{a}{b}</div>` to `jsxs('div', {children: [a, b]})` and `<div>{a}</div>` to `jsx('div', {children:a})`. The `jsxs` function indicates that the top array was created by React. The nice property of this strategy is that even if you don't have separate build steps for PROD and DEV, we can still issue the key warning and still pay no cost in PROD.

### DEV only transforms

We have special transforms meant only for DEV. `__source` and `__self` are not part of `props`. We can pass them as separate arguments instead.

One possible solution is to compile DEV as a separate function:

```js
jsxDEV(type, props, key, isStaticChildren, source, self)
```

That way we can easily error if the transform doesn't match.

### Spread only

This particular pattern:

```js
<div {...props} />
```

Can currently be safely optimized to:

```js
createElement('div', props)
```

That's because `createElement()` always clones the passed object. We want to avoid cloning in the `jsx()` function with the new transform. Most of the time this won't be observable because the JSX creates a new inline object anyway. This is a special case where it doesn't.

We could solve this by always cloning inline:

```js
jsx('div', {...props})
```

Or, we could leave it like this:

```js
jsx('div', props)
```

This would be a breaking change, but we could always clone in the call in a minor and then make the breaking change later in a major. The new semantics would be that the passed in object gets frozen (in DEV).

## Deprecate "module pattern" components

```js
const Foo = (props) => {
  return {
    onClick() {
      //...
    }
    render() {
      return <div onClick={this.onClick.bind(this)} />;
    }
  }
};
```

It causes some implementation complexity just by existing.

This is pretty straight forward to upgrade from. This is a very unusual pattern and most people don't know you can. The key is that your class constructor need to have a `Component.prototype.isReactComponent` property and handle being called with `new` (i.e. no arrow functions). Even if you happen to use the module pattern you can add a prototype with a `isReactComponent` property and use function expression instead of arrow function.

```js
function Foo(props) {
  return {
    onClick() {
      //...
    }
    render() {
      return <div onClick={this.onClick.bind(this)} />;
    }
  }
};
Foo.prototype = {isReactComponent: true};
```

The important goal here is that if we're going to introduce different semantics between classes and function components, we need to know before calling them which semantics we're going to apply.

## Deprecate `defaultProps` on function components

`defaultProps` is very useful on classes because the props object gets passed to many different methods. Life-cycles, callbacks etc. Each one in its own scope. This makes it hard to use JS default arguments because you'd have to replicate the same defaults in each function.

```js
class Foo {
  static defaultProps = {foo: 1};
  componentDidMount() {
    let foo = this.props.foo;
    console.log(foo);
  }
  componentDidUpdate() {
    let foo = this.props.foo;
    console.log(foo);
  }
  componentWillUnmount() {
    let foo = this.props.foo;
    console.log(foo);
  }
  handleClick = () => {
    let foo = this.props.foo;
    console.log(foo);
  }
  render() {
    let foo = this.props.foo;
    console.log(foo);
    return <div onClick={this.handleClick} />;
  }
}
```

However, in function components there really isn't much need for this pattern since you can just use JS default arguments and all the places where you typically use these values are within the same scope.

```js
function Foo({foo = 1}) {
  useEffect(() => {
    console.log(foo);
    return () => {
      console.log(foo);
    };
  });
  let handleClick = () => {
    console.log(foo);
  };
  console.log(foo);
  return <div onClick={handleClick} />;
}
```

We'd add a warning in createElement if something that doesn't have `.prototype.isReactComponent` on it uses `defaultProps`. This includes other special components like `forwardRef` and `memo`.

It's trickier to upgrade if you're passing the whole props object around but you can always reconstruct it if needed:

```js
function Foo({foo = 1, bar = "hello"}) {
  let props = {foo, bar};
  //...
}
```

## Deprecate spreading `key` from objects

This pattern is currently supported:

```js
let randomObj = {key: 'foo'};
let element = <div {...randomObj} />;
element.key; // 'foo'
```

The problem with this is that we can't statically know if this object is going to pass a key or not. So for every set of props, we have to do an expensive dynamic property check to see if there is a `key` prop in there.

My proposal is that we solve this by treating a static `key` prop as different from one provided through spread. I think that as a second step we might want to even give separate syntax such as:

```js
<div @key="Hi" />
```

To minimize churn and open up a larger discussion about this syntax, we'd instead treat `key` as a keyword in JSX and pass it separately.

The backwards compatible implementation of `jsx(...)`, we would still support `key` passed as props. We'd just pull it off props and issue a warning that this pattern is deprecated. The upgrade path is to just pass it to JSX separately if you need it.

```js
let {key, ...props} = obj;
<div key={key} {...props} />
```

An unresolved issue is how we distinguish `<div key="Hi" {...props} />` from `<div {...props} key="Hi" />` which currently have different semantics depending on if props has a `key`.

In a later major, we'd stop extracting key from props and therefore props is now just passthrough.

## Deprecate string refs (and remove production mode `_owner` field).

We know we want to deprecate string refs. We already warn about this in strict mode. It's time to start warning about it in general.

In a future major, we'll remove string refs and that will let us get rid of the `_owner` field from elements.

We have a transform that adds `__self`. We can use this to issue different warnings when `__self` and `_owner` have the same value. In those cases it is safe to run an automated codemod of string refs from `ref="foo"` to `ref={n => this.refs.foo = n}`. Therefore the recommendation becomes to first fix all the cases where `__self` and `_owner` are *different* since those need manual intervention. That warning can go out earlier. After that warning has taken effect, we can next tell people to run a codemod for the rest.

## Move `ref` extraction to class render time and `forwardRef` render time

In a minor, we'll add an enumerable getter in DEV for `props.ref` if a ref is defined on its element. This will warn if you try to access it. However, in class components, we'll detect this and create a copy of the props before passing it into the class. The same thing applies to `forwardRef`. We can also special case `cloneElement`.

Since you can't pass a ref to anything but host, class and forwardRef, it should be fairly unusual that you're spreading props that has a ref on it. Hopefully it is possible to work around the remaining cases.

In the next major, we'll start copying the ref onto both the props and the `element.ref`. React will now use the `props.ref` as the source of truth for `forwardRef` and classes and it will still create a shallow copy of props that excludes the ref in these cases. At the same time, we'll add a getter for `element.ref` in DEV that warns if you access it. The upgrade path is now to just access it off props if you need it from the element.

## Move `defaultProps` resolution to class render time

In a minor, after we've already deprecated `defaultProps` for non-classes, we'd start adding getters for all props that were resolved using `defaultProps` in DEV. Before passing this object to a class, we'd do a shallow clone and pass the props without the getters. These getters issue a warning, that you're reading from props of an element early on. Maybe `cloneElement` gets a special pass.

The upgrade path here, is to just avoid reading from `element.props` or move away from relying on `defaultProps` and passing them in explicitly or resolving them in the class.

In the next major, we'd stop resolving `defaultProps` during element creation and instead, we'd only resolve it right before we pass them into class components.

# Drawbacks

The main drawback here is that there is some manual work to upgrade user code. The type of things that needs to be changed are unusual patterns but they may be spread out. We can add warnings that track most patterns so if you have sufficient logging tools to follow up on these few edge cases, upgrades should be managable.

It also puts churn on the surrounding tooling ecosystem - like type systems.

There might also be a slight performance cost during the transition period.

# Alternatives

An alternative would be to leave this in place for now and then try to address it more holistically as a larger compiler project that might include more changes as well.

# Adoption strategy

Early on we'd have to deploy any changes to JSX transforms since it takes a long time for these to be deployed and they're typically not versioned together with React. The implementation would be backwards compatible.

In a minor version close in time to an expected major release we would include warnings for the deprecated patterns and instructions for how to change them to one that doesn't warn.

In the next major version, we would actually move ref/defaultProps resolution to classes. forwardRef would extract it from props, but now we could soft-deprecate forwardRef and instead just recommend pulling it off of props.

# How we teach this

The hard part is teaching the upgrade path. Once done, the result is significantly simpler since we remove many concepts. Especially if you teach function components first.

# Unresolved questions

- How do we distinguish `<div key="foo" {...props} />` from `<div {...props} key="foo" />` during the upgrade path stage?
