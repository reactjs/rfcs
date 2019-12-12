- Start Date: (2019-12-04)
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

This proposal introduces the concept of signals and slots used in Qt to
React.

Signals and slots would be extremely useful for React.
It will solve the ongoing problem of child-to-child communication and will
also lead to less tightly-coupled components.

# Basic example

```js
import { useRef, useSignal, connect } from 'react';
import PropTypes from 'prop-types';

function Button ({ name }) {
    const emit = useSignal('clicked');
    
    const onClick = () => emit(`Hello, ${name}!`);

    return (
        <button onClick={onclick}>{name}</button>
    );
}

Button.signals = {
    clicked: PropTypes.string
};

function Banner ({ message }) {
    return (
        <h1>{message}</h1>
    );
}

Banner.propTypes = {
    message: PropTypes.string
};

function App () {
    const buttonRef = useRef(null);
    const bannerRef = useRef(null);
    connect(buttonRef, 'clicked', bannerRef, 'message');
    
    return (
        <>
            <Banner ref={bannerRef} />
            <Button ref={buttonRef} name="World" />
        </>
    );
}
```

# Motivation

Child-to-child communication is notoriously difficult in React, but it's also
inherent to React. It should be solvable using React, not something that forces
developers to rely on external libraries. Signals and slots could solve the
problem of child-to-child communication in a natural, React-like way.

Because the signals and slots API is implemented using React Hooks, it shares
the same benefits of being **completely optional** and
**100% backwards-compatible**.

# Detailed design

The signals and slots concept in Qt has been one of the most successful in the
history of software engineering. It's typesafe, decoupled, extensible, and
*fun*! It enables components to communicate with their parents and siblings,
without being tightly coupled to either.

In Qt, slots are methods defined in a C++ class, and signals are functions
defined in a class but are implemented by a preproccessor, not the developer.
At first glance, it might seem like adapting that to React would require a lot
of extra code since React components are just functions.
But React components already have a mechanism for slots: JSX props.
Instead of slots being functions as they are in Qt, slots are parameters.

The `connect` syntax is really where things get difficult.
The syntax Qt uses is something like
`connect(MyTimer, QTimer::fired, MyWindow, WindowClass::update)`. Obviously
that can't be directly ported to React.

The primary reason for requiring any signals or slots to be defined at
compile-time is because they have to be implemented using Qt's
meta-object preprocessor and compiler.
In React, specifying signals and slots as string names means that all
components continue to work as-is without breaking anything.
It also has the benefit of backwards compatibility: New components that emit
signals can be connected to components that haven't been updated in years.

In this proposal:  
*Signals* are events that are defined by React components using the `useSignal`
hook.  
*Slots* are React props that have been *connected* to a signal using the
`connect` hook.

Above all, I wish to stress: **Signals and Slots is not for state-management.
It is for inter-component communication.**

# Drawbacks

Most of the drawbacks of React Hooks also apply to here.
A non-exhaustive list of drawbacks specific to this proposal include:

- May require considerable changes to the internals of React.
- Might be difficult to implement a proof-of-concept outside of React.
- `connect()` might need to be implemented by the renderer (ReactDOM) rather
  than React itself.
- If documentation is unclear it could easily lead to confusion about how this
  is used and what to use it for.
- Additional prop-types are needed for signals in order to guarentee type safety.
- The current design uses strings as params for `connect()`, which makes
  type-checking with TypeScript/Flow much more difficult.
- The same end result (albiet at a cost) could be achieved with existing
  userspace state-management libraries (Redux) or global event emitters.
- Signals and slots is not suitable for sending large amounts of data or a
  large number of items.
- The current `useRef` syntax may cause unexpected behavior because refs
  are mutable.

# Alternatives

A number of alternate syntaxes were considered.

One alternative considered was:
```js
const [ signal, slot ] = connect('clicked', 'message')
<Button signal={signal} />
<Banner slot={slot} />
```
Problem with that approach is that it doesn't connect components, and
requires more thorough changes to the internals of React.
It also sacrifices typesafety and comes at a cost of being much more verbose.
I also considered having slot types be specified separate from regular
propTypes (`Banner.slots = { message: ... }`), but I realized that was
redundant and lead to duplication of information.

My original proposal used referencing `const MyButton = Button` to connect
components. That has been removed because the internals of React make the
intended behavior impossible.

I originally planned to have a `useSlot()` hook, but decided that went against
the concept of signals and slots. The whole purpose is to send messages between
components, not synchronize state. Besides, that means that only new components
would be able to benefit from the signals and slots API. The current syntax
proposal enables signals to be connected even to old components.

Current solutions to the problem of inter-component communication may use:
* Callbacks-as-params
* Global state
* React context API.
* Third-party state-management libraries.
* Global event managers.

In my opinion, none of these is suitable for solving the use-cases this
proposal intends to solve.
All have significant overhead in both code size and complexity.
Most are procedural, not declarative, and are also very difficult
to connect to external component libraries.
The developer must specify the behavior of not only the components
that are connecting to each other, but also the behavior that connects
the components together.
This leads to inherent coupling between the sender, the receiver, and the
connector.

# Adoption strategy

The beauty of this is that nothing breaks even if a signal or a slot is not
implemented.
Remove the `useSignal` code from Button or the `message` prop from Banner, and
both will continue to work just fine.

Once implemented, developers can start adopting it at their own pace.
Library maintainers can add it to their packages without breaking anything, and
then just simply list the signals emitted in the documentation.
Developers can then start using those signals at their own leisure.

Signals can immediately be added to any React component without change.
Developers can then start migrating away from any state-management solutions
because existing param names can continue to be used as-is,
or new params can be added as needed.

# How we teach this

This should be thought of as the natural evolution of functional-style React.
Slots are just regular props for a React component, and signals are just
events/messages defined by a React component.

All existing React concepts still apply, just like all other React Hooks.
No major changes to the React documentation are needed, and learning how to
use it is completely optional for new developers.
For existing developers, instead of defining *how* components should
communicate, they now define *what* components are communicating to each other.

# Unresolved questions

How to avoid undesired connections (i.e. you have two Banner components but you
only want one connected to 'clicked').
My above proposal uses the ref property defined by `useRef()`.
This might cause problems because refs are mutable.

How to handle type-safety in Flow and TypeScript.
The fact that signals are defined using a React Hook, and the connection syntax
uses strings to specify signal and slot names much static type-checking much
more difficult.

Related to that is the `Component.signals = {...}` syntax.
Although React might still be able to use this information to enable internal
type-safety, if type-checking cannot be implemented in Flow or TypeScript this
might be removed.

If you have an alternative syntax for the signals and slots concept, feel free
to discuss it.
