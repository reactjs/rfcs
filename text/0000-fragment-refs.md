- Start Date: 2018-12-10
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

An alternative way to findDOMNode to drill into a set of children rendered by abstractions.

# Basic example

```js
function Foo({children}) {
  let fragmentRef = useRef();
  useEffect(() => {
    let domNodes = fragmentRef.current;
    // ...
  });
  return <Fragment ref={fragmentRef}>{children}</Fragment>;
}
```

# Motivation

We [deprecated `findDOMNode(...)` in StrictMode](https://github.com/facebook/react/pull/13841) because it's mostly unnessary and has a number of flaws that means that it should be avoided.

- `findDOMNode(...)` can be called at arbitrary times so we need to be able to query the tree for the "current" version of any given thing. This is a tough ask in a concurrent environment. As a result, the current implementation is slow in some cases and filled with hacks to make it slightly faster in some cases.
- `findDOMNode(...)` can be called on any class component at any time. Therefore we can't optimize away trees or intermediate components just in case something needs to reference them to support this API.
- `findDOMNode(...)` doesn't work with functional components and hooks since there is not "instance" to refer to.
- `findDOMNode(...)` only returns a single item, but since we can now return strings and fragments, there could be more than once child.
- `findDOMNode(...)` is called at a point when you assume there is a child but you can't always know when the child swaps. Such as when an intermediate component renders two different children. E.g. `c ? <div /> : <span />`. It doesn't provide a way to react to this change like callback refs do.

Another motivation for deprecating this API was also that you can use it to drill through abstractions which is typically a bad use of the API since if you mutate those children, you might conflict with the actual owners use of those children.

However, there are some use cases where you're not mutating children but querying them. A common example is to read the layout of a child. You don't necessarily care about the particular node itself but the boundary of the child.

There is a way to solve all of these by adding an additional wrapper node. You can even use the wrapper to access the children:

```js
function Foo({children}) {
  let fragmentRef = useRef();
  useEffect(() => {
    let domNodes = fragmentRef.current.children;
    // ...
  });
  return <x-fragment ref={fragmentRef}>{children}</x-fragment>;
}
```

There are some quirks with real DOM nodes as wrapper nodes though which is why we avoid them and have the virtual fragments instead. E.g. it can affect layout. It's not valid HTML in all positions such as in tables and various lists.

Therefore this proposal is to expose a kind of ref that gives you access to a virtual fragment instance just as if it was a real DOM node.

# Detailed design

The concrete proposal here is that we'd enable a `ref` to be passed to a `<Fragment />` component. When mounted or the list of children within this fragment changes, the callback ref would be invoked with an Array of those children. This Array is conceptually a fragment node. When it unmounts, it gets passed null.

```js
function Foo({children}) {
  function handleRef(childNodes) {
    if (childNodes === null) {
      // unmounted
    } else {
      // mounted, reordered, or removed nodes
      for (let node of childNodes) {
        // node is a DOMNode
      }
    }
  }
  return <Fragment ref={handleRef}>{children}</Fragment>;
}
```

The child nodes are always DOMNodes because it drills through any custom component and finds the DOM nodes. It never includes classes instances or anything using forwardRef.

This avoids a number of issues with the `findDOMNode` design:

- It would have a callback that fires when any of them changes so you would have a way to detect when the child components changes. E.g. when they swap or get added/removed/reordered.
- It only needs to know what the "current" children are during the commit phase. Not at arbitrary random times which makes the implementation much faster and more predictable.
- It's opt-in so React doesn't need to store or emit trees to support these for everything in the whole tree - just in case.
- It works with function components.
- It supports more than one child node (such as when fragments are returned from render).

# Drawbacks

This design still exposes a way to drill into child nodes. This means that components are not fully modular in the sense that a parent can mess with them in unpredictable ways. That is already technically the case since the DOM exposes traversal APIs but it's not encouraged. Arguably this API almost encourages abuse.

There is some extra work involved in React to keep track and issue updates to the ref each time a child set changes. In some environments this happens anyway but not always.

# Alternatives

The alternative to this API is simply to not add anything and instead encourage libraries to use custom elements as wrappers. E.g. `x-fragment`. These can use tricks like `display: contents` to play nicer into layouts. Ideally the DOM would support something like this first class.

I'd say that a custom element wrapper node is almost always technically strictly better than virtual fragments for this use case. The ownership model is clear. You only reference the parent, never the nested child, so the modularity is clear. You don't have to respond to child changes so performance is better. Whenever you reorder or move a set of children, performance is better since you only do one move instead of moving each child inside the fragment. The cost of additional DOM nodes in modern browsers is widely exaggerated.

That said, there is some loss of "seamlessness" due to the quirks mentioned above, so not supporting some kind of seamless fragment is an uphill battle again library authors wanting to provide somekind of seamless API to their consumers.

# Adoption strategy

This API can't really be feature tested since we warn on refs. So ideally libraries would just cut a version that depends on a newer version of React instead of trying to support findDOMNode and this API at once. Especially since these libraries can't give guarantees about supporting fragments which is best practice anyway.

# How we teach this

TBD

# Unresolved questions

TBD