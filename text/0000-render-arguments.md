- Start Date: 2017-12-08
- RFC PR:
- React Issue:

# Summary

Add arguments for the component's render method.

# Basic example

```js
class SomeComponent extends React.Component {

    // Props are passed as the first argument of the render method
    // State is passed as the second argument
    render(props, state) {

        return (
            <div className={state.selected}>
                {props.data}
            </div>
        );
    }
}
```

# Motivation

`.render()` method is the only required method of react components, it always
uses the component props and/or state, so there is the need to always write the
`this.props` or `this.state` code to get access to the data that is needed.
This proposal is about passing the `props` and `state` as arguments to the
`.render()` method, which will allow a cleaner structure of the most used
component method.

A side effect of this approach is that it will allow an easy
migration from functional components to class components by moving the function
content to the render method of the class without any major changes.

# Detailed design

The implementation is quite simple and obvious for implementation, this section
can be added later in case it's needed.

# Drawbacks

A drawback of this approach is the possible misuse of render arguments in
closures created during the execution of `.render()` method. The following code
can be used as an example:

```js
render(props, state) {

    return (
        <div onClick={() => doSomething(props.data, state.selected)}></div>
    );
}
```

In this case, when the click event happens it might be that the actual props or
state of the components have changed since the last render but the values used
inside the inline event handler are stale and can cause some undesired behavior
of the component.

This can be bypassed by avoiding the creation of closures inside the `.render()`
method, which should be a recommendation for this approach.

# Alternatives

An alternative way to access `props` or `state` would be to provide
a single object as an argument to the `.render()` method which contains the
two containers, in this way the required properties can be accessed using
destructuring like in the example below:

```js
render({ props, state }) {

    return (
        <div className={props.className}>{state.label}</div>
    );
}
```

# Adoption strategy

This proposal should not break any code, moreover react-like libraries have this
approach implemented.

# How we teach this

This approach should be taught in the docs as an extension of the `.render()`
method documentation. It should be presented as an alternative for the `this.`
approach, but with less code to write and a limitation. The limitation consists
in the usage of the cached arguments values in closures that are created inside
the `.render()` method. This approach can also be presented as another reason to
always use the well-known best practice in react to avoid creating inline
functions for component props and avoid like this some unnecessary re-renders.

# Unresolved questions

A decision should be made between the two alternative approaches, one with two
arguments `.render(props, state)` and the other one with an argument containing
the two containers as properties and accessible using destructuring
`.render({ props, state })`.