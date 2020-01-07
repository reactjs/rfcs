- Start Date: 2020-01-07
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Reduce the amount of magic behind Hooks by modifying Hooks API and making it
more straightforward.

# Basic example

```jsx
import React, { useState } from 'react';

// Functional component receives second optional parameter - component instance.
// It makes more obvious where useState hook stores state variables.
function Example(props, instance) {
  // Declare a new state variable, which we'll call "count".
  const [count, setCount] = useState(0, instance);

  return (
    <div>
      <p>{props.userName} clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>
        Click me
      </button>
    </div>
  );
}
```

# Motivation

Hooks were introduced in React 16.8. It's an interesting API that allows using
state and other React features without writing a class. The main idea of Hooks
is
[reducing components complexity](https://reactjs.org/docs/hooks-intro.html#complex-components-become-hard-to-understand)
by splitting a component into several functions. It is said that
[classes confuse both people and machines](https://reactjs.org/docs/hooks-intro.html#classes-confuse-both-people-and-machines)
because you have to understand how `this` works in JavaScript and you have to
remember to bind the event handlers before using them.

Before Hooks, functional components were
[pure functions](https://en.wikipedia.org/wiki/Pure_function), easy to
understand and test. Pure functions return the same result for the same
arguments. The functional component with Hooks works in a different way: even if
the component is called with the same `props` it can return different values
because of different data stored in the internal React state associated with the
component.  The idea that `useState` returns one result if functional component
is called for the first time for some instance in virtual DOM and another
result if it called again, is very confusing because the state is hidden
somewhere and that is why sometimes it's preferable to understand how `this`
works in class components and write a bit more verbose but clear code than to
debug magic of functional components with Hooks.

The only way how current behavior of `useState` can be implemented is some
hidden global variable and it's better to avoid using global variables when it's
possible.

It will be very useful to make Hooks API a bit more straightforward.

# Detailed design

Functional components have such API:

```jsx
function MyComponent(props) {
  // ...
}
```

It's proposed to change it to this one:


```jsx
function MyComponent(props, instance) {
  // ...
}
```

Main Hooks have the following signatures:

* `useState(initialState)`
* `useEffect(create, inputs)`

It's proposed to change it to these ones:

* `useState(initialState, instance)`
* `useEffect(create, inputs, instance)`

The new `instance` parameter is a virtual DOM instance currently worked on (like
`this` in class components). It makes more obvious for the users where the state
is stored and what component is currently being processed. It also makes testing
more straightforward because the functional component depends only on input
parameters and no hidden global state:

```jsx
const instance = prepareInstance();
const result = MyComponent(props, instance);
// check the result
```

It also makes React code more robust especially for concurrent rendering because
there are no global variables that can be corrupted by another Fiber (see
[React Fiber Architecture](https://github.com/acdlite/react-fiber-architecture)
for more details).

This feature can be implemented without breaking changes: just make `instance`
parameter optional for current major React version and if the user calls
`useState` without second parameter use currently implemented behavior but show
a warning in the console.

# Drawbacks

As any code changes of React framework, it costs some time for implementation
and testing and also it can lead to new bugs. If the users will update to the
new major version of React they have to change their code.

# Alternatives

TBD

# Adoption strategy

This feature can be implemented without breaking changes: just make `instance`
parameter optional for current major React version and if the user calls
`useState` without second parameter use currently implemented behavior but show
warning in console. For the future versions of React calling `useState` or
`useEffect` without additional parameter should lead to an error.

It's possible to help users change existing code with some tools like codemod.

# How we teach this

* Hooks documentation should be altered
* A warning should be implemented for the current major version of React
* Blog post with explanation why such changes are important
