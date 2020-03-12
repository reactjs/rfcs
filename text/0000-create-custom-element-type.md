- Start Date: 2018-01-22
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

The changes in this RFC specifically address how React passes data to custom
elements and listens for their DOM events.

# Basic example

This RFC has two parts.

1. React would change its current behavior such that when it passes data to a
   custom element, it uses JavaScript properties instead of calling
   `setAttribute`.

For example:

```
<my-element fooBar={baz}>
```

Would become equivalent to:

```
myElement.fooBar = baz;
```

2. React would add a new API, `ReactDOM.createCustomElementType()` which would
   create a wrapper React component that knows how to map properties and event
   handlers back to a custom element.

# Motivation

These two changes are meant to address
[#7249](https://github.com/facebook/react/issues/7249),
[#7901](https://github.com/facebook/react/issues/7901),
[#11347](https://github.com/facebook/react/issues/11347) and a handful of other
related issues. The goal is to make it easier to integrate [custom
elements](https://developers.google.com/web/fundamentals/web-components/customelements)
in React projects.

As shown on [Custom Elements
Everywhere](https://custom-elements-everywhere.com/#react), using a custom
element in React today requires a handful of workarounds to pass complex
JavaScript data (objects, and arrays) to custom element properties or listen for
their DOM events.

# Detailed design

We're proposing React make both of these changes:

## 1. Prefer JavaScript properties on Custom Elements

By default, setting a prop on a custom element will use a JavaScript property
setter. This changes React's current behavior, which uses `setAttribute()`.

For example:

```jsx
<my-element fooBar={baz}>
```

Would become equivalent to:

```jsx
myElement.fooBar = baz;
```

When calling `ReactDOMServer.renderToString()`, props on custom elements should
be set as attributes. Camel case props should be converted to all lowercase
attributes.

Example:

```jsx
const baz = 'hello';
<my-element fooBar={baz}>      // before renderToString()
'<my-element foobar="hello">'  // after renderToString()
```

Note: Some custom element libraries use a heuristic where camel case properties
are converted to dash-cased attributes. E.g. `.fooBar` becomes `foo-bar=""`.
Prior experience has shown that this actually confuses a fair number of
developers. Plus there is prior art in HTML for doing all lowercase attributes,
e.g. `contenteditable`, `autocomplete`, `tabindex`. For these reasons, we think
it's easiest to convert camel cased properties to lowercase attributes, without
a delimiter, when calling `ReactDOMServer.renderToString()`.

## 2. ReactDOM.createCustomElementType()

Developers can use the new `createCustomElementType()` API to export a wrapper
React component which knows how to map props to the underlying custom element.
Because of the changes outlined in the first point, this wrapper would only be
necessary in situations where:

- an element only exposes an attribute API with no corresponding property.
- during SSR, a property name cannot easily be mapped back to an attribute name
  by lowercasing it.
- serializing a property value to an attribute value requires a custom function,
  e.g. calling `JSON.stringify()`.
- you want to be able to declaratively listen for DOM events from a custom
  element (currently not supported in React,
  [#7901](https://github.com/facebook/react/issues/7901)).

Example signature:

```js
ReactDOM.createCustomElementType(tagName[, configuration])
```

Example configuration:

```js
export default ReactDOM.createCustomElementType('x-foo', {
  propName: {
    // Attribute name to map to.
    attribute: string | null,

    // Serialize function to convert prop value
    // to attribute string.
    serialize: function | null,

    // Indicates the prop is an event, and the value
    // should be an event handler function.
    // If the event key is used, then all of the other
    // keys (property, attribute, serialize) should throw
    // compile warnings if set.
    event: string | null
  }
  propName2: {
    ...
  }
  ...
});
```

Example usage:

```jsx
// XFoo.js
export default ReactDOM.createCustomElementType('x-foo', {
  longName: {
    attribute: 'longname'
  },
  someJSONdata: {
    attribute: 'somejsondata',
    serialize: JSON.stringify
  },
  onURLChanged: {
    event: 'urlchanged'
  }
});
```

```jsx
// App.js
import XFoo from './XFoo';
import React from 'react';
import {render} from 'react-dom';

function App() {
  const name = 'Ronald McDonald';
  const data = {hello: 'world'};
  const handleChange = function() {
    console.log('the url changed!');
  }

  return (
    <XFoo longName={name}
          someJSONdata={data}
          onURLChanged={handleChange} />
  )
}

render(<App />, document.querySelector('#app'))
```

# Drawbacks

- Changing to setting properties instead of calling `setAttribute()` on custom
  elements would be an API breaking change.
  - However, developers can use `ReactDOM.createCustomElementType()` to get
    things back in working order.
  - Or, if the attribute name matches their property name, things will "just
    work".

# Alternatives

There has been an immense amount of discussion of alternatives at
[#11347](https://github.com/facebook/react/issues/11347). The current proposal
was arrived at with the React team and the community after working through about
5 different contenders. For custom element authors, switching to a
properties-first approach would be very convenient for them and unlock the
ability to easily pass objects and arrays to custom elements in JSX. For folks
who want an additional layer of security, or just prefer to look of working with
React components throughout their app, they could use the
`ReactDOM.createCustomElementType()` API.

One possible area where we could tweak things would be map camelCased properties
to dash-cased attributes, instead of going all lowercase. Because all Polymer
elements follow this heuristic (and many/most(?) custom elements are written in
Polymer), it would mean that they should all continue to "just work" with no
code changes.

# Adoption strategy

**If we implement this proposal, how will existing React developers adopt it?**

This would only affect the subset of React developers who also use custom
elements. If their custom elements expose a JS properties interface (which is
recommended in [Google's best practices
docs](https://developers.google.com/web/fundamentals/web-components/best-practices#always-accept-primitive-data-strings-numbers-booleans-as-either-attributes--or-properties))
then it may require no code changes at all. However, if they don't have
corresponding properties for their attributes, they will either need to: update
their element, or update their React app to explicitly call `setAttribute()`, or
use the proposed `ReactDOM.createCustomElementType()` API.

**Is this a breaking change?**

Yes but we estimate that the number of users affected would be small.

**Can we write a codemod?**

Someone could probably write a codemod specific to their component to update it
in all of their apps.

**Should we coordinate with other projects or libraries?**

I can't think of a reason why that would be necessary, but perhaps others have
opinions :)

# How we teach this

There are already docs on [using Web Components in
React](https://reactjs.org/docs/web-components.html), so we could update them to
explain these changes. We would also need to update the `ReactDOM` API docs to
explain the new `createCustomElementType()` API. Probably would want to link to
the Web Components docs from that API explainer.

Again, we imagine this should only affect a small subset of the React community,
those folks currently trying to integrate custom elements into their apps.
