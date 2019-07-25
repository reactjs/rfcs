- Start Date: 2018-10-25
- RFC PR: https://github.com/reactjs/rfcs/pull/68
- React Issue: (leave this empty)

# Summary

In this RFC, we propose introducing *Hooks* to React.

* Hooks let you **reuse logic between components** without changing your component hierarchy.
* Hooks let you **split one component into smaller functions** based on what pieces are related.
* Hooks let you **use React without classes**.

Hooks are opt-in and **100% backwards-compatible**. You can use Hooks side by side with your existing code.

*This RFC is very detailed and inconvenient to read as a single Markdown file. Instead of condensing it and sacrificing important details, we decided to write our proposal in the form of documentation, and link to individual pages below. You can consider this documentation to be a part of the RFC, so please feel free to quote and discuss any content from it when commenting on the RFC.*

# Basic example

This example renders a counter. When you click the button, it increments the value:

```js
import { useState } from 'react';

function Example() {
  // Declare a new state variable, which we'll call "count"
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>
        Click me
      </button>
    </div>
  );
}
```

This rest of this section is covered by:

* **[Hooks at a Glance](https://reactjs.org/docs/hooks-overview.html)**

# Motivation

This section is covered by:

* **[Introducing Hooks: Motivation](https://reactjs.org/docs/hooks-intro.html#motivation)**

# Detailed design

This section is covered by:

* **[Using the State Hook](https://reactjs.org/docs/hooks-state.html)**
* **[Using the Effect Hook](https://reactjs.org/docs/hooks-effect.html)**
* **[Rules of Hooks](https://reactjs.org/docs/hooks-rules.html)**
* **[Writing Custom Hooks](https://reactjs.org/docs/hooks-custom.html)**
* **[Hooks API Reference](https://reactjs.org/docs/hooks-reference.html)**
* **[Hooks FAQ](https://reactjs.org/docs/hooks-faq.html)**

Some subtle aspects of this design that we especially appreciate:

* Values returned from one Hook can be passed to another. For example, an effect can read the return value from `useState` because it is automatically in function scope.
* Custom Hooks allow abstracting logic without any special involvement from React.
* One variable per `useState` call instead of a single `this.state` object means fewer object property accesses that are hard for VMs to optimize.
* State variables names (with `useState`) can be automatically minified.
* No changes to build tooling are required.

# Drawbacks

A non-exhaustive list of drawbacks of this Hooks design follows.

* Introducing a new way to write components means more to learn and means confusion while both classes and functions are used.
* The “Rules of Hooks”: in order to make Hooks work, React requires that Hooks are called unconditionally. Component authors may find it unintuitive that Hook calls can't be moved inside `if` statements, loops, and helper functions.
* The “Rules of Hooks” can make some types of refactoring more difficult. For example, adding an early return to a component is no longer possible without moving all Hook calls to before that conditional.
* Event handlers need to be recreated on each render in order to reference the latest copy of props and state, which reduces the effectiveness of `PureComponent` and `React.memo`.
* It's possible for closures (like the ones passed to `useEffect` and `useCallback`) to capture **old versions** of props and state values. In particular, this happens if the “inputs” array is inadvertently missing one of captured variables. This can be confusing.
* React relies on internal global state in order to determine which component is currently rendering when each Hook is called. This is “less pure” and may be unintuitive.
* `React.memo` (as a replacement for `shouldComponentUpdate`) only has access to the old and new props; there's no easy way to skip rerendering for an inconsequential state update.
* `useState` uses a tuple return value that requires typing the same name twice to declare a state field (like `const [rhinoceros, setRhinoceros] = useState(null);`), which may be cumbersome for long names.
* `useState` uses the overloaded type `() => T | T` to support lazy initialization. But when storing a function in state (that is, when `T` is a function type) you must always use a lazy initializer `useState(() => myFunction)` because the types are indistinguishable at runtime. Similarly, the functional updater form must be used when setting state to a new function value.

# Alternatives

Possible alternatives follow. In our opinion, none of these cleanly solve all the problems that Hooks do.

* The status quo: supporting state and effects only in class components, with no changes.
* Alternative designs that allow using state and effects without classes:
    * Language-level syntax akin to `state foo = 5;` in the style of [DisplayScript](http://displayscript.org/).
    * Built-in render prop components that provide stateful features, like [Reactions Component](https://github.com/reactions/component).
    * Built-in higher-order components [that pass stateful features as arguments](https://mobile.twitter.com/acdlite/status/971598256454098944).
    * Other “stateful function components” ideas laid out in [react-future](https://github.com/reactjs/react-future).
* Alternative designs that facilitate sharing of stateful logic:
    * Syntax sugar for render props, like [an `adopt` keyword](https://gist.github.com/trueadm/17beb64288e30192f3aa29cad0218067) or [Epitath](https://medium.com/astrocoders/epitath-in-memoriam-render-props-and-hocs-9f76dd911f9e).
    * Reintroducing [mixins in class components](https://reactjs.org/blog/2016/07/13/mixins-considered-harmful.html).

# Adoption strategy

We're planning to recommend a [gradual adoption strategy](https://reactjs.org/docs/hooks-intro.html#gradual-adoption-strategy) for Hooks. Hooks don't break existing patterns, and they can be used side by side with class components. It's possible that codemod could be written to automatically convert simple class components to use Hooks instead, but none is currently planned.

# How we teach this

The documentation linked above is our best attempt at teaching this. Over time we expect that we'll integrate the Hooks documentation with the main concept docs instead of having it in a separate section.

# Credits and prior art

Hooks synthesize ideas from several different sources:

* Our old experiments with functional APIs in the [react-future](https://github.com/reactjs/react-future/tree/master/07%20-%20Returning%20State) repository.
* React community's experiments with render prop APIs, including [Ryan Florence](https://github.com/ryanflorence)'s [Reactions Component](https://github.com/reactions/component).
* [Dominic Gannaway](https://github.com/trueadm)'s [`adopt` keyword](https://gist.github.com/trueadm/17beb64288e30192f3aa29cad0218067) proposal as a sugar syntax for render props.
* State variables and state cells in [DisplayScript](http://displayscript.org/introduction.html).
* [Reducer components](https://reasonml.github.io/reason-react/docs/en/state-actions-reducer.html) in ReasonReact.
* [Subscriptions](http://reactivex.io/rxjs/class/es6/Subscription.js~Subscription.html) in Rx.
* [Algebraic effects](https://github.com/ocamllabs/ocaml-effects-tutorial#2-effectful-computations-in-a-pure-setting) in Multicore OCaml.

[Sebastian Markbåge](https://github.com/sebmarkbage) came up with the original design for Hooks, later refined by [Andrew Clark](https://github.com/acdlite), [Sophie Alpert](https://github.com/sophiebits), [Dominic Gannaway](https://github.com/trueadm), and other members of the React team.
