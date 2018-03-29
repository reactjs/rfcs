- Start Date: 29 March 2018
- RFC PR: 
- React Issue: 

# Summary

JSX will be transpiled to object literals, native types to factory functions.

# Basic Example

```jsx
<div>
Hello, World!
</div>
```

Will be transpiled to:

```jsx
{
  type: ReactDOM.div,
  props: {
    children: "Hello, World!"
  }
}
```

Instead of:

```jsx
React.createElement('div', {}, 'Hello, World!')
```

Which resolves to:

```jsx
{
  $$typeof: Symbol("react.element"),
  type: "div",
  props: {
    children: "Hello, World!"
  },
  key: null,
  ref: null,
}
```

# Motivation

 - No more JSX pragma: React-like libraries like Preact would be able to consume React components with **no synthetic syntax modifications**. Of course there can be many other differences but there is enough common ground that worths the changes in my opinion.

 - Allows greater composabillity and easier modification of elements as they are just POJOs.
 
 - Clearer line between ReactDOM components and renderer as types are now factory functions that can validate props themselves
 
 - (Maybe) Reduced render time because React.createElement is not called N times every render
 
# Detailed design

### Security

Unlike in previous discussion this proposal is secure because **type is always a function**.
Unlike strings a function can't be injected by XSS or similar. If a JSON will be parsed type will remain string and will be considered an error:
```json
{ "type": "div", "props": { "children": "Hello, World!" } }
```
Any object not conforms to the exact shape of React element will be considered error as well:
```typescript
type Element = {
  type: React.ComponentType<*>,
  props: Object,
  key?: any,
  ref?: (null | Object) => void,
}
```

# Possible Drawbacks

 - `React.createElement()` is required to determine element's owner efficently
 - `ReactDOM` would have to be imported for DOM elements creation which creates a new required pragma - can be discussed further.

# Adoption strategy

Babel plugin would be updated to the new syntax.
The element APIs (`React.createElement()`, `React.Children`, `React.cloneElement()`...) will be modified to work with the new format.

# How we teach this

- Simplifies the public API of React to the point a component is just a function / class returning a POJO.
  Example like this can make sense for a lot of people:

```javascript
const MyComponent = () => ({
  type: ReactDOM.div,
  props: {
    children: 'Hello, World!'
  }
})
```

# Unresolved questions

 - How will consumers use their own DOM elements? (`@jsx` pragma for elements)
 - How does it affect rendering
