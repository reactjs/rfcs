- Start Date: 2025-08-12
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

A way to expose some React internals so it becomes easier to develop tools to debug and interact with React applications.

# Basic example

A global variable like `__REACT_DEVTOOLS_GLOBAL_HOOK__` but it is meant to be used by other tools and not just React Devtools.

# Motivation

Why are we doing this?

To allow other developers create tools like [react-scan](https://www.reddit.com/r/reactjs/comments/1h70kfg/reverse_engineering_react_internals/) without running the risk of having your code broken at any time.

What use cases does it support?

Developers have to test their code constantly with each react release regardless of whether it is a release that introduces major breaking changes or minor backward compatible ones. It would be nice to introduce an api with it's own versioning scheme that we can rely on.

What is the expected outcome?

An API that exposes some basic functionality that might be useful to create tools that interact with React applications separately, such as manipulating the state and props some components are rendered with, or maybe forcing rerenders or remounting.

# Detailed design

The global variable may be an object with a property "version" that represent the version of this api. This version is different from the version of react because a minor react version can introduce breaking changes in this api. Developers that rely on this api then would refer to some documentation that explains how to use this api in this specific version.

This global variable might, for instance, have an "onLoad" callback function that would be called by react once react is fully loaded just in case someone wanted to rely on it but couldn't make sure to have their code executed after react. Then you'd have to check if this global object is available and has a "version" property to know if react is loaded to start using the api and if not then you'd create this object with your "onLoad" callback.

We may also add some functions to this global object to interact with react:
- `getState(component, stateIndex?)`: Change the state of the component of the first argument. `stateIndex` refers to the nth call of `useState()`. If omitted, it will return the state of all calls of `useState()` in an array.
- `setState(component, stateIndex, newValue)`: Change the state of the component.
- `resetState(component, stateIndex?)`: Reset the state to its default value. If no `stateIndex` is given, reset the state of all `useState()` calls.
- `rerender(component)`: Force a rerender.
- `remount(component)`: Force a remount. Basically, call all `useEffects()` again even if their dependencies didn't change. Useful, for instance, if a component makes a request to load the data that it displays and you want to reload it without refreshing the page or having to run the logic that would unmount and remount it.
- `getReactRoot()`: Get the root react element. You can traverse the children of the root component to get the reference that you'd pass to the other functions.

# Drawbacks

Why should we *not* do this? Please consider:

- implementation cost, both in term of code size and complexity: depends on how much we want to expose. If we only wanted to allow developers to change the state of components, maybe not much.
- whether the proposed feature can be implemented in user space: it can with tools like [bippy](https://github.com/aidenybai/bippy), but it's a matter of time that they break because it relies on react internals that don't have any kind of api or versioning.
- the impact on teaching people React: this change wouldn't affect how people learn to use react, but would help greatly those who have to reverse engineer the source code to make tools that change the state of a component externally like React Devtools does.
- integration of this feature with other existing and planned features: we'd have to consider whether it would be worth it to expose some api to interact with new features from now on.
- cost of migrating existing React applications (is it a breaking change?): it's not a breaking change, so zero.

# Alternatives

What other designs have been considered?

[Bippy](https://github.com/aidenybai/bippy)

What is the impact of not doing this?

There is this self-explanatory warning in the bippy's README:

> [!WARNING]
> ⚠️⚠️⚠️ **this project may break production apps and cause unexpected behavior** ⚠️⚠️⚠️
>
> this project uses react internals, which can change at any time. it is not recommended to depend on internals unless you really, _really_ have to. by proceeding, you acknowledge the risk of breaking your own code or apps that use your code.

# Adoption strategy

If we implement this proposal, how will existing React developers adopt it? They will have to refactor the part of their code that relies on react internals that are exposed by this api.

Is this a breaking change?

No

Can we write a codemod?

No

Should we coordinate with other projects or libraries?

No

# How we teach this

What names and terminology work best for these concepts and why? How is this idea best presented? As a continuation of existing React patterns?

TBD

Would the acceptance of this proposal mean the React documentation must be re-organized or altered?

Yes, we need to document this api somewhere.

Does it change how React is taught to new developers at any level?

No

How should this feature be taught to existing React developers?

TBD

# Unresolved questions

Optional, but suggested for first drafts. What parts of the design are still TBD?

We need to decide what to name this global variable and what to include in the first versions.

