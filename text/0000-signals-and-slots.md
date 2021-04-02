- Start Date: (2019-12-04)
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

This proposal introduces the concept of signals and slots to React.

Signals and slots are used in the Qt C++ framework to implement the observer
pattern without needing any boilerplate code. It enables components to be
completely decoupled from the internal state or interface of other components.
Adding this to React will also decouple components from their parent component
while also isolating the internal state of each component.

# Basic example

```js
import React, { useSignal, useSlot, useState } from 'react';
import PropTypes from 'prop-types';

function TextInput () {
    const [ value, setValue ] = useState('');
    const emit = useSignal('updated');
    const onChange = (e) => {
        setValue(e.target.value);
        emit(e.target.value);
    };
    return <input value={value} onChange={onChange}
}

TextInput.signals = {
    updated: PropTypes.string
};

function Banner({ name }) {
    return <h1>Hello, {name}!</h1>;
}

Banner.propTypes = {
    name: PropTypes.string
};

function App () {
    const name = useSlot('updated');
    
    return (
        <>
            <Banner name={name} />
            <TextInput />
        </>
    );
}
```

# Motivation

Modifying the state of another component in React inherently leads to tightly
coupled components. The most widespread practice today is to use `useState()` or
`useReduce()` and then pass both the values and the callbacks to modify those
values to a child component.

That exposes the internal state of each component, and it leads to all
components being dependent on each other's interface. Change even one parameter
name, and the whole thing can fall apart.

Signals and slots can solve the problem of tightly coupled components in a
natural, React-like way.

Because the signals and slots API is implemented using React Hooks, it is
completely optional and backward-compatible.

# Detailed design

The signals and slots concept in Qt has been one of the most successful in the
history of software engineering. It's typesafe, decoupled, extensible, and
*fun*! It enables components to communicate with their parents and siblings
without being tightly coupled to each other.

In Qt, slots are methods defined in a C++ class, and signals are functions
defined in a class but are implemented by a preprocessor, not the developer.
The React implementation differs from the Qt framework in that it connects
children to parents instead of child-to-child.

This design decision was made because, unlike C++, React does not provide
an easy way of keeping track of an individual instance of a child component.
Fortunately, React does provide an easy way of providing the exact same thing:
JSX props.

A parent can connect to a signal and then pass the value as a prop to any
child component. That enables defacto child-to-child communication without
any extra boilerplate code needed.

In this proposal:  
*Signals* are events that are defined by React components using the `useSignal`
hook.  
Slots are values assigned by listening for a signal using the `useSlot` hook.

Above all, I wish to stress: **Signals and Slots is not for state management.
It is for inter-component communication.**

# Drawbacks

Most of the drawbacks of React Hooks also apply here.
A non-exhaustive list of drawbacks specific to this proposal include:

- May require considerable changes to the internals of React.
- Might be difficult to implement a proof-of-concept outside of React.
- If documentation is unclear, it could lead to confusion about how this
  is used and what to use it for.
- Additional prop-types are needed for signals in order to guarantee type safety.
- The current design uses strings as params for `useSlot()`, which makes
  type-checking with TypeScript/Flow much more difficult.
- The same end result (albeit at a cost) could be achieved with existing
  userspace state-management libraries (Redux) or global event emitters.
- Signals and slots is not suitable for sending large amounts of data or a
  large number of items.
- If multiple children emit the same signal name, the parent component has no
  way of knowing which child emitted it.

# Alternatives

A number of alternate syntaxes were considered.

The most recent proposal used child-to-child connections but required the use
of refs to keep track of each individual child. That was discarded for being
too complex and difficult to understand.

One alternative considered was:
```js
const [ signal, slot ] = connect('clicked', 'message')
<Button signal={signal} />
<Banner slot={slot} />
```
The problem with that approach is that it doesn't connect components, and
requires many more changes to the internals of React.
It also sacrifices type-safety and comes at the cost of being much more verbose.
I also considered having slot types be specified separate from regular
propTypes (`Banner.slots = { message: ... }`), but I realized that was
redundant and lead to duplication of information.

My original proposal used referencing `const MyButton = Button` to connect
components. That has been removed because the internals of React makes the
intended behavior impossible.

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
Remove the `useSignal` code from Button or the `useSlot` value from App, and
although the behavior will not be as desired, all components will continue
to work.

Once implemented, developers can start adopting it at their own pace.
Library maintainers can add it to their packages without breaking anything and
then just list the signals emitted in the documentation.
Developers can then start using those signals at their own leisure.

Signals can immediately be added to any React component without change.
Developers can then start migrating away from any state-management solutions
by capturing signals using `useSlot()` and passing the value to a component.

# How we teach this

This should be thought of as the natural evolution of functional-style React.
Slots are just regular values that can be passed to a React component, and
signals are just events/messages defined by a React component.

All existing React concepts still apply, just like all other React Hooks.
No major changes to the React documentation are needed, and learning how to
use it is completely optional for new developers.
For existing developers, instead of defining *how* components should
communicate, they now define *what* components are communicating to each other.

Another benefit is that this proposal could lead to better documentation for
library maintainers. JSX props should be documented like input parameters
in a procedural language, and signals can be documented as being output
parameters.

# Unresolved questions

How to handle multiple components emitting the same signal name.
If a parent has two components that emit the same signal (e.x. CancelButton
and SubmitButton both emit "clicked"), then parents have no way of knowing
which child emitted it.

How to handle type-safety in Flow and TypeScript.
Slots should be easy enough to add type-safety to. But signals are much more
difficult. Connecting the `Component.signals = {...}` props to `useSlot()`
could pose several problems.

If you have an alternative syntax for the signals and slots concept, feel free
to discuss it.
