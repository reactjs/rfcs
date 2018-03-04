- Start Date: 2018-03-03
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Provide a way of registering a handler for a custom prop that is passed as a prop name using computed property syntax like `[customPropHandler]={value}` and acts as ref on the property value instead of a named prop.

The intent is to allow implementation of props that can provide custom events, custom attributes, and other behaviours which are simple and act on a single prop value but require the use of a complex `ref` in order to work.

# Basic example

Custom events are not the only you can do with registered props as refs. However they are a good example use case that is useful and requires complex ref handler behaviour to work.
```js
import React, {CustomPropRegistry} from 'React';
import ReactDOM from 'react-dom';

const PropRegistry = new CustomPropRegistry();

// Register an ref prop that binds a my-custom-event event handler and stores it in a onMyCustomEvent variable
const onMyCustomEvent = PropRegistry.registerRefProp((ref, prevRef, value, prevValue) => {
  if ( prevRef && prevValue && (ref !== prevRef || value !== prevValue) ) {
    prevRef.removeEventListener('my-custom-event', prevValue);
  }

  if ( ref && value && (ref !== prevRef || value !== prevValue) ) {
    ref.addEventListener('my-custom-event', value);
  }
});

// Use the onMyCustomEvent variable as a prop name
const MyComponent = () => (
    <custom-element [onMyCustomEvent]={(e) => { console.log(e); }} />
);

// Render, providing the propRegistry as an option
ReactDOM.render(<PropRegistry.Provider><MyComponent /></PropRegistry.Provider>, container);
```

# Motivation

React has had long-standing issues with handling of custom attributes, custom elements, and custom events.

Custom attributes were fixed by loosening the attribute whitelist. Custom elements behaviour is [under active discussion](https://github.com/facebook/react/issues/11347). And there is still debate on how to handle [custom events in web components](https://github.com/facebook/react/issues/7901).

These issues bring up how complex the properties/attributes/events associated with a prop can be. A string can be either the name of an attribute or a property and it's difficult to decide which it should be (`amp` must be an attribute, but `value` must be a property). A prop name could also be a request to register an event. But which props names should be events? Should we assume every event name is lowercase and turn `onValueChanged` to `value-changed`, even though it's possible a custom event may use upper case characters like `DOMContentLoaded` and vendor events like `MozOrientation`? What about capturing events? How do you define an event listener with the new `passive` option?

These issues will likely be solved eventually, but they all seem to suggest that at least on the web there are use cases for props on DOM elements that go beyond what React can do with simple string based key props. Currently the answer to many of these issues is "define your attributes/events imperatively using a ref prop". But ref props are too complex for simple definitions of attributes/events. They work imperatively and are normally larger than a simple declaration. As a result of this and PureComponent ref function are normally defined outside of props, moving what should be prop definitions out of render().

I believe this RFC is not mutually exclusive with the fixes for the custom attributes and custom events issues. We can still add custom event behaviours like `domEvents`. Even if we fix all the normal custom attributes/properties and events issues, it is still possible there may be special or advanced cases that they do not cover. Or certain ecosystems may work more optimally if they were able to register ref props instead of relying on built-in attribute and event handling syntax.

# Detailed design

### CustomPropRegistry

A `CustomPropRegistry` class is exposed from the `'react'` package, the class contains a "symbol => handler" registry of ref prop handlers.

In environments that do not support `Symbol` it may not be necessary for an entire `Symbol` polyfill to be included, instead a sufficiently unique and possibly random prefix to an incrementing integer should be enough to separate the result of registerRefProp from normal string props. Including a character not valid in a JSX Attribute name such as "!"" or "|" in the prefix would also make it difficult for the prefix to match an actual prop name.

### CustomPropRegistry#registerRefProp((ref, prevRef, value, prevValue) => void) => symbol

`registerRefProp` creates a Symbol and associates it with the handler in the registry and then returns the symbol.

### CustomPropRegistry#Provider

When instantiated a wrapped `Provider` component is created for the registry. This Provider component works similarly to the `createContext` API's `Provider`. However instead of accepting a `value` the Provider merely provides the `CustomPropRegistry` instance. And there is no `Consumer`, instead registries are provided internally to all components below the provider for use in internal element implementations.

### <some-element [refProp]={value} />

`refProp` refers to the symbol returned from `registerRefProp`. The `[refProp]={value}` is not custom syntax for this RFC but a recommendation that the JSXAttribute syntax is extended to support ES2015's computed property names.

If `<some-element prop={value} />` were to expand to `React.createElement('some-element', {prop: value})` then `<some-element [refProp]={value} />` would expand to `React.createElement('some-element', {[refProp]: value})` and act as it would in ES2015.

### Handling of ref props

React does not handle ref props itself. If a ref prop is passed to a React Component it does not behave like `ref`, instead the ref prop will be available on that component's prop such that it would be returned by `props[refProp]` given that `refProp` refers to the same symbol as used in `[refProp]={value}`.

Ref props are handled in ReactDOM for elements that refer to dom nodes. For every prop name that is present in the registry the prop is not included in the normal dom props/attributes/events handling, instead the handler registered with `registerRefProp` is called under certain changes to the node.

A handler takes the following arguments `ref, prevRef, value, prevValue`. It is called with the following arguments in the following cases:

- On creation/mount of the node / when the ref prop is first added to a node:
  ```js
  handler(node, null, propValue, null)
  ```
- When the value of the prop changes (`nextProps[refProp] !== prevProps[refProp]`):
  ```js
  handler(node, node, newPropValue, prevPropValue)
  ```
- When the node is removed/unmounted / when the prop is removed from the node:
  ```js
  handler(null, node, null, propValue)
  ```
- I am not aware of any situation where the dom node associated with a component is replaced with another one. But if it were then the handler would be called like this:
  ```js
  handler(newNode, prevNode, propValue, propValue)
  ```

This pattern provides enough information to do the following:
- When `ref && (ref !== prevRef)` "setup" actions like inserting dom elements can be run
- When `prevRef && (ref !== prevRef)` "cleanup" actions like removing elements/handlers can be run
- `value !== prevValue` can be used for simple change operations like `ref.property = value;`

Conditions that also cover multiple actions are also possible:
- `ref && (ref !== prevRef || value !== prevValue)` covers "setup" and "value change"
- `prevRef && (ref !== prevRef || value !== prevValue)` covers "value change" and "cleanup"
- `ref && value && (ref !== prevRef || value !== prevValue)` can be used to run "setup" operations on the `ref` and `value` when the ref is added or value is changed, but not if the `value` is *falsey*. Like setting up an event handler, but not if the handler is given a *falsey* value instead of a function to temporarily disable the handler.
- `prevRef && prevValue && (ref !== prevRef || value !== prevValue)` can be used to run "cleanup" operations on the `prevRef` and `prevValue` when the ref is removed or value is changed, but not if the `prevValue` was *falsey*. Like cleaning up an event handler.

# Drawbacks

- The `[refProp]={...}` syntax requires either pre-registration or an inline function call to a memoized function. Even if it's a helpful advanced feature custom attribute/property issues are best solved in other ways and the other custom event proposals do not all require custom registration.
- If JSX is updated to support computed property name / computed JSX Attribute name syntax it may be possible for wrappers like [skatejs/val](https://github.com/skatejs/val) to implement this in user-space.
  - The library would have to implement a registry and registration function,
  - React's `createElement` would need to be wrapped and a `@jsx` comment used in all files using JSX,
  - props names in the registry would need to be intercepted,
  - and a computed `ref` would be needed along with `componentDidUpdate` to check for prop changes in order to make the necessary calls to the handler.
  - Because the registry is not part of react the library we may end up with multiple implementations. Any library implementing common use cases like registering custom events would need to prefer one wrapper or implement support for multiple ones.

# Alternatives

## Handler interface

Separate handlers like `{add: handler, change: handler, remove: handler}` were considered as instead of the `ref, prevRef, value, prevValue` interface. However events, properties, and attributes have different requirements and typically need to run the same code for multiple types of operations. An interface with an `action` argument was also considered, but would likely result in code like `if ( action === "prop-add" || action === "prop-change" )` which would not be reliable if new actions were added and would still additionally require the same set of of ref/value/prev/next props for both cleanup and setup operations to work.

## Alternative registry location

A ref prop registry internal to ReactDOM where `CustomPropRegistry#registerRefProp` would instead be `ReactDOM.registerRefProp` was considered. However this had drawbacks if there were multiple instances of React/ReactDOM or no longer needed the registered props and wanted to reclaim memory.

Passing registries as an option to the `ReactDOM.render` function was also considered. However a whole new options argument may not be warranted if we can use Providers. Providers also leaves the door open for CustomPropRegistry to be used in environments other than `react-dom`.

# Adoption strategy

- Coordination with JSX implementations will be necessary. The spec will need to be updated to add ES2015's computed property names to JSXAttribute. And implementations will need to be updated to convert a computed JSX property to an ES2015 computed property.
- This is an additional feature, it does not remove or change existing behaviour so it should not be a breaking change.
- When Symbol is not available a unique string is enough for ref props to work. It should not be necessary to make a breaking change that raises React's minimum environment to one that includes a Symbol polyfill.

# How we teach this

JSX's syntax already contains a spread operator that matches ES2015's spread. It should be sufficient to teach the ES2015 computed property syntax in relation to JSX the same way as we teach JSX's spread operator.

The `CustomPropRegistry` API itself will likely be considered an advanced API. Instead of directly teaching the API to users it would likely be more useful to create libraries for common use cases like custom events and teach usage of those to people.

# Unresolved questions

- In "Handling of ref props" it's uncertain what the optimal algorithm for handling refProps would be.
  a. Iterate over all Symbol property names on `props`, check the registry for each one and process the prop as a ref prop instead of a normal one if found. When Symbol is not available do the same but instead on keys that match the string prefix used by the library.
  b. Iterate over all ref prop names registered, if it is present in `props` process it as a ref prop instead of a normal prop.
- For isomorphic web apps and code shared between React DOM / React Native versions of an app it may be beneficial to provide some way to make a `refProp` that is a no-op, so the same template can be used in multiple environments with no-ops replacing ref props that work on dom nodes.
- Should we use `React.createCustomPropRegistry()` instead of `new CustomPropRegistry()` to match the new Context API a little closer?
- Should `'react'` expose an internal/unstable helper to handle ref props in a component's props? This would allow other environments such as React Native to choose to permit the same kind of handling for components like `View` that directly expose native elements.

# Sample libraries

## react-create-event

This is a simple library that exposes a memoized `createEvent` that can be used to quick and easily register custom events to use in prop names.

### react-create-event/index.js
```js
import React, {CustomPropRegistry} from 'React';
const EventPropRegistry = new CustomPropRegistry();

export const Provider = EventPropRegistry.Provider;

const events = Object.create(null);
export default function createEvent(eventName) {
  // memoize so the result of createEvent does not need to be stored
  if ( !events[eventName] ) {
    events[eventName] = EventPropRegistry.registerRefProp((ref, prevRef, value, prevValue) => {
      if ( prevRef && prevValue && (ref !== prevRef || value !== prevValue) ) {
        prevRef.removeEventListener(eventName, prevValue);
      }

      if ( ref && value && (ref !== prevRef || value !== prevValue) ) {
        ref.addEventListener(eventName, value);
      }
    });
  }

  return events[eventName];
};
```

### Examples
```js
import React from 'react';
import createEvent, {Provider} from 'react-create-event';

const onMyCustomEvent = createEvent('my-custom-event');

const eventHandler = () => {};
const MyComponent = () => (
  <custom-element [onMyCustomEvent]={eventHandler} />
);

ReactDOM.render(<Provider><MyComponent /></Provider>, container);
```

Or it could be used without pre-registering events:
```js
import e from 'react-create-event';

const eventHandler = () => {};
const MyComponent = () => (
  <custom-element [e('my-custom-event')]={eventHandler} />
);
```
