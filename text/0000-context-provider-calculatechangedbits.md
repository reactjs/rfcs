- Start Date: 2018-08-25
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

`Context.Provider` should be updated to accept a `calculateChangedBits` function as a prop, either in addition to or as a replacement of giving `calculateChangedBits` as an argument to `React.createContext`.

# Basic example

```js
import React from "react";

const ContextA = React.createContext();
const ContextB = React.createContext();

function calculateChangedBitsForArray(currentValue, nextValue) {
    // magic logic to diff arrays and generate a bitmask
}

function calculateChangedBitsForPlainObject(currentValue, nextValue) {
    // magic logic to diff objects and generate a bitmask
}

class App extends React.Component {
    state = { array : ["a", "b", "c"], object : {a : 1, b : 2, c : 3} }
    
    render() {
        return (
            <ContextA.Provider 
                value={this.state.array}
                calculateChangedBits={calculateChangedBitsForArray}
            >
				<ContextB.Provider 
					value={this.state.object}
					calculateChangedBits={calculateChangedBitsForPlainObject}
				>
				    {this.props.children}
				</ContextB.Provider>
			<ContextA.Provider>
        )
    }
}
```

# Motivation

### End Goal

The end goal is to enable flexible customization of context update handling at runtime, as determined by the code that renders the `<Context.Provider>`

### Background

React 16.3 introduced the new `React.createContext` API, which is intended to be a stable version of "context".  `createContext` returns an object with a `Provider/Consumer` pair of components.  When a `<Context.Provider value={someValue} />` is rendered higher in a tree, instances of that `<Context.Consumer>` can be rendered deeper in the tree to access the latest value.

By default, React itself does an initial check on the Provider's `value` prop to see if it has changed.  If the value given to the Provider hasn't changed, then React will not cause the associated rendered Consumer instances to update.

The documentation says that "context uses reference identity to determine when to re-render".  More specifically, the current implementation compares the previous and current values using an inlined version of the [`Object.is()` comparison](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/is), which does both a reference check and checks for several edge cases such as nulls or `NaN`.

If the previous and current values have changed, React then tries to generate a 31-bit integer that acts as a bitmasked value.  If the bitmask value is 0, then it is also assumed no context Consumer updates are needed.  If the bitmask is non-zero, React will check each Consumer's `unstable_observedBits` prop if available, and only update that consumer if at least one of the requested "observed bits" has been marked as updated.  

By default, React marks _all_ bits in the bitmask as having changed, causing all Consumers to update.  However, `createContext` can take a (currently undocumented) argument called `calculateChangedBits`, which is a function with a signature of `(oldValue, newValue) => changesBitmask`.  This allows end users to potentially optimize which Consumers are actually updated, by only setting certain bits in the bitmask based on the value changes.  (The bitmask values themselves have no specific meaning - it's up to the end user to determine how they might be calculated, such as hashing string key names to a bit index.)


### Use Case

The current design is not flexible enough.  Most context use cases likely involve `Context.Provider/Consumer` pairs being treated as singletons - instantiated once in an app or library, and used everywhere.  It would be very useful to customize the changed bits calculation for a given `Provider` whenever it is rendered, but that is impossible with the current API - `calculateChangedBits` can only be passed in to `createContext`, which means that it will be used _everywhere_ that singleton `Context.Provider` is rendered.

Here's a specific example.  A typical Redux app has a state tree that is made up of plain JS objects and arrays, and the root of the tree is a plain object.  Assuming that React-Redux has been refactored to use `createContext`, it would be reasonable to define a React-Redux-specific `calculateChangedBits` implementation that compares the keys and values of two plain objects, and for any changed keys, hashes the keys into consistent bits to generate the bitmask.  Assuming that connected components had some way of indicating which state keys they rely on, that would allow a component that only cares about `state.a` to completely skip the updating process if only `state.b` was updated.

However, currently that `calculateChangedBits` function has to be passed into the `createContext` call, and React-Redux would likely use a singleton `ReactReduxContext.Provider/Consumer` pair everywhere.  That means that if a Redux app uses something else for its root state (such as an array, an Immutable.js `Map`, or some other specialized value), the default `calculateChangedBits` implementation wouldn't work right (and would in fact likely break).

If React's `Context.Provider` supported defining `calculateChangedBits` as a prop, then a React-Redux end user could define their own custom `calculateChangedBits` function specific to their own app's state, and pass that to the React-Redux `<Provider>` when they render their app.  

Ultimately, the end result should be that it is possible to define the changed bits calculation process when a `Context.Provider` is rendered, and even change that process if necessary.


# Detailed design

This is the bulk of the RFC. Explain the design in enough detail for somebody
familiar with React to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

### Current Implementation

- `React.createContext` [accepts a `calculateChangedBits` function as its second argument](https://github.com/facebook/react/blob/53ddcec4f18f38e4f89a14b406d852e7a8945592/packages/react/src/ReactContext.js#L32-L48)
- The `calculateChangedBits` function [is stored as a field in a `ReactContext` object](https://github.com/facebook/react/blob/53ddcec4f18f38e4f89a14b406d852e7a8945592/packages/react/src/ReactContext.js#L50-L64)
- When [React begins updating a context provider](https://github.com/facebook/react/blob/master/packages/react-reconciler/src/ReactFiberBeginWork.js#L911-L915), it calls the [internal `calculateChangedBits](context, newValue, oldValue)` function](https://github.com/facebook/react/blob/53ddcec4f18f38e4f89a14b406d852e7a8945592/packages/react-reconciler/src/ReactFiberNewContext.js#L108-L139)  and [passes in the current context, new value, and old value](https://github.com/facebook/react/blob/master/packages/react-reconciler/src/ReactFiberBeginWork.js#L943).
- Inside the internal `calculateChangedBits`, it [checks to see if the context instance was given a `calculateChangedBits` function](https://github.com/facebook/react/blob/53ddcec4f18f38e4f89a14b406d852e7a8945592/packages/react-reconciler/src/ReactFiberNewContext.js#L125-L127), and uses it if available.  The `context._calculateChangedBits` lookup is the only use of the `context` argument in that function.


### Possible Updated Implementation

I _think_ this could be done with just a few lines in three files.  All that's needed is to try to use the provided prop function if it exists, and directly pass the user-provided function instead of the internal context object.

In `ReactFiberBeginWork.js`, function `updateContextProvider`:

```diff
  if (oldProps !== null) {
    const oldValue = oldProps.value;
+   const _calculateChangedBits = newProps.calculateChangedBits || context._calculateChangedBits;
-   const changedBits = calculateChangedBits(context, newValue, oldValue);
+   const changedBits = calculateChangedBits(_calculateChangedBits, newValue, oldValue);
```

In `ReactFiberNewContext.js`, function `calculateChangedBits`:

```diff
export function calculateChangedBits<T>(
- context: ReactContext<T>,
+ _calculateChangedBits : function | undefined
  newValue: T,
  oldValue: T,
) {
  // Use Object.is to compare the new context value to the old value. Inlined
  // Object.is polyfill.
  // https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/is
  if (
    (oldValue === newValue &&
      (oldValue !== 0 || 1 / oldValue === 1 / (newValue: any))) ||
    (oldValue !== oldValue && newValue !== newValue) // eslint-disable-line no-self-compare
  ) {
    // No change
    return 0;
  } else {
    const changedBits =
-     typeof context._calculateChangedBits === 'function'
-       ? context._calculateChangedBits(oldValue, newValue)
+     typeof _calculateChangedBits === 'function'
+       ? _calculateChangedBits(oldValue, newValue)
        : MAX_SIGNED_31_BIT_INT;
```

Finally, it appears that `Context.Provider` has a `PropTypes` declaration somewhere, so that would need to be updated to accept `calculateChangedBits : PropTypes.function`.

# Drawbacks

I see no meaningful drawbacks to adding this capability:

- The implementation cost is potentially just 4 changed lines and 2 added lines
- This is not a feature that can be implemented in user space as far as I can see
- It's a heavily advanced (and thus far even undocumented) feature that is really only intended for library usage, so there's no need to teach this to people learning React
- Assuming that the "changed bits" functionality is going to be kept around, it only makes that functionality more flexible without changing how it works
- The user-facing change is an additional accepted prop on `Context.Provider`, which would not break any existing user code.



# Alternatives

In briefly thinking about it, the only other alternative userland implementation I've come up with for a use case like React-Redux would be to somehow have React-Redux's `<Provider>` generate unique instances of a React `Context.Provider/Consumer` pair so that it could pass a user-provided `calculateChangedBits` function to `React.createContext`, then pass those down using the singleton `ReactReduxContext.Provider`.  A nested child component would have to render a `ReactReduxContext.Consumer`, grab out the unique generated `Context.Consumer` instance, and render _that_ to get the actual value out.  It might be feasible, but it's a very twisted workaround.



# Adoption strategy

The primary users of this would be state management libraries like React-Redux, Mobx-React, React-Copy-On-Write, etc.  Informal notice that the API has been updated would be sufficient, along with perhaps formally documenting the change (and possibly making `unstable_observedBits` a stable API).



# How we teach this

No "teaching" work needs to be done here.

