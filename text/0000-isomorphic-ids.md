- Start Date: 2018-03-08
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Many accessibility features of HTML require linking elements using IDs. There is
currently no easy and consistent way to programmatically handle these IDs in a 
way that survives the client hydration process when server-side rendering is 
involved. 

The key issue to solve here is a way to express relationships between 
HTML elements that are identified by the ID attribute in a way that allows that 
relationship to be valid and deterministic on both the server-side and 
client-side render of the tree.

# Basic example

Consider a simple Checkbox component.

```js
// Checkbox.js
class Checkbox extends React.Component {
  
  render() {
    const { label, value, name, disabled } = this.props;
    const id = generateUniqueId(); // Any UUID-generating function.
    
    return (
      <div>
        <input
          id={id}
          type="checkbox"
          name={name}
          value={value ? value : label}
          disabled={disabled || false}
        />
        <label htmlFor={id}>{label}</label>
      </div>
    );
  }
  
}

// App.js
<Checkbox label="My Label" value="my-label" name="MyLabel" disabled={false} />
```

The current state of the isomorphic world would result in something to the 
effect of:

```
// Browser console.
Warning: prop `id` did not match. Server: "Checkbox:..." Client: "Checkbox:...."
```

```html
<!-- Server output -->
<div>
  <input
    id="unique-id-1"
    type="checkbox"
    name="MyLabel"
    value="my-label"
  />
  <label htmlFor="unique-id-1">My Label</label>
</div>

<!-- Client output -->
<div>
  <input
    id="unique-id-3"
    type="checkbox"
    name="MyLabel"
    value="my-label"
  />
  <label htmlFor="unique-id-3">My Label</label>
</div>
```

**The goal of this RFC is to ensure that there is a way to guarantee that the 
IDs used by the server and client are the same.**

# Motivation

In order to successfully link the `label` with the `input`, the `input` element
must have an ID and that same ID must be supplied to the `label` in the `html-for`
attribute.

From this component's perspective, the relationship between these two elements 
is always encapsulated inside the component. If only considering client-side 
rendering, the simple solution would be to generate a random, unique ID, store 
it as an instance variable, and assign that value to both elements.

This procedure breaks down when introducing server-side rendering because the 
value will be different between the server's execution and the client's execution
resulting in an invariance violation (see the Basic Example above).

To work around this, the developer must break encapsulation and supply a unique
value for every instance. This introduces additional human input and puts an 
unnecessary strain on developers to consider and coordinate uniqueness of IDs 
for these types of components.

There should be no need for users of this component to consider global uniqueness
when the component itself should know everything it needs to know to render 
correctly.

# Detailed design

There are probably several ways that this could be addressed. The simplest from a 
conceptual standpoint would be to ensure that the render path is deterministic so
that an auto-increment ID generation approach would result in the same number 
being assigned to each ID.

This is not a great solution. First off, it will probably result in a significant
performance penalty which would be unacceptable. Secondly, it does not take into 
account potential (intentional) differences in what is rendered server-side and 
client-side.

A more robust solution would be to take advantage of the hydration process to 
absorb attributes included in the server-rendered output into the hydrated DOM 
before it is merged into the browser's DOM.

Let us suppose that we had a way to reserve an identifier from React. Something
akin to the following:

```js
// Checkbox.js
class Checkbox extends React.Component {
  
  constructor() {
    this.reserveUniqueIdentifier(); // <------------------------------------NEW
  }
  
  render() {
    const { label, value, name, disabled } = this.props;
    const id = this.reservedIdentifier(); // <----------------------------- NEW
    
    return (
      <div>
        <input
          id={id}
          type="checkbox"
          name={name}
          value={value ? value : label}
          disabled={disabled || false}
        />
        <label htmlFor={id}>{label}</label>
      </div>
    );
  }
  
}

// App.js
<Checkbox label="My Label" value="my-label" name="MyLabel" disabled={false} />
```

On the server, this component would now be rendered as:

```html
<div data-react-reserved-id="6CB96365-C679-415A-8FDE-8CD52C2AA678">
  <input
    id="6CB96365-C679-415A-8FDE-8CD52C2AA678"
    type="checkbox"
    name="MyLabel"
    value="my-label"
  />
  <label htmlFor="6CB96365-C679-415A-8FDE-8CD52C2AA678">My Label</label>
</div>
```

> NOTE: Calling `this.reservedIdentifier` without (or before) calling 
> `this.reserveUniqueIdentifier` should result in an exception being thrown.

By calling `this.reserveUniqueIdentifier` in the constructor, an additional 
property can be added to the object to tag it as having a reserved ID. This will
be useful later for quickly determining if additional steps are needed during 
hydration.

Example:
```js
const checkbox = <Checkbox label="My Label" value="my-label" name="MyLabel" disabled={false} />;
console.log(checkbox.hasReservedUniqueIdentifier);
// => true
```

During hydration, whenever a component is encountered that has a reserved ID, 
it is scheduled for an update before the shadow DOM is merged with the browser's
DOM. That element's position in the tree is determined and the corresponding 
element in the browser's DOM is retrieved, the `data-react-reserved-id` attribute
is read, and its value is set as the reserved ID for the hydrated component. 
Then that component is enqueued for an update. Finally, after all components with
reserved IDs have been updated, the hydrated DOM can be merged with the browser's 
DOM in the usual fashion.

In the case where a corresponding element is not found in the browser's DOM 
(meaning that is was omitted during server-side render), then a new unique ID 
is generated and that new ID is used by the hydrated component.

Furthermore, since this process is part of the hydration process then these 
additional checks will not be performed if not intentionally hydrating SSR output
ensuring that there is no impact on the `render` path.

### Step-by-step procedure

#### Server procedure

1. `ReactDOM.renderToStaticMarkup` is called.
1. A `Checkbox` component is encountered.
1. The `Checkbox` constructor calls `this.reserveUniqueIdentifier()`.
1. `reserveUniqueIdentifier` determines that it is running on the server using
   `fbjs/lib/ExecutionEnvironment`.
1. A unique ID is generated (for example using `UUIDv4`) and set to a private
   property on the instance. (for example `this.__reservedIdentifier`)
1. Property `hasReservedUniqueIdentifier` is set to `true` on the instance.
1. `Checkbox` component's `render` method is called.
1. The reserved ID is retrieved by the `this.reservedIdentifier()` method
   that retrieves the value of the private instance variable.
1. The `data-react-reserved-id` HTML attribute is added to the root element 
   of the component with the value of `this.reservedIdentifier()`.
1. HTML is sent to the client.

#### Client procedure

1. `ReactDOM.hydrate` is called.
1. A data structure is created to store references to elements that have reserved
   identifiers.
1. A component where `hasReservedUniqueIdentifier` is `true`.
1. A reference to the Component instance is stored in the above data structure.
1. After the whole client DOM has been processed:
  1. Iterate over instances stored in the above data structure.
  1. Determine the DOM path of the element in the shadow DOM.
  1. Look up the equivalent DOM path in the browser's DOM.
  1. If an element is found:
    1. Retrieve the value of `data-react-reserved-id`.
  1. If an element is not found:
    1. Generate a new ID.
  1. Set the value of the instance's `__reservedIdentifier` private property.
  1. Enqueue that instance for an update.
  1. Perform queued in the shadow DOM.
1. Merge shadow DOM with browser DOM in normal fashion.
    
#### Alternative client procedure

1. `ReactDOM.hydrate` is called.
1. A component where `hasReservedUniqueIdentifier` is `true`.
1. Look up the equivalent DOM path in the browser's DOM.
1. If an element is found:
  1. Retrieve the value of `data-react-reserved-id`.
1. If an element is not found:
  1. Generate a new ID.
1. Set the value of the instance's `__reservedIdentifier` private property.
1. Continue processing the tree.
1. Merge shadow DOM with browser DOM in normal fashion.
    

# Drawbacks

The major drawback with this approach is that it introduces some potentially 
expensive DOM queries to match up the server-generated DOM with the hydrated 
client DOM as well as an extra step during hydration that causes updates to be 
made before the first client DOM update is made. To be successful, this approach 
should not make a noticeable impact on performance.

The design outlined above is intended to have a very light impact on the existing
architecture but due to my lack of familiarness with the internals of React, there 
will undoubtedly be issues that I have not anticipated.

# Alternatives

Currently available techniques are lacking since they all require the client code
to be merged with the DOM before queries can be made to extract any attributes
that have been left by the server-rendered output. This results in either lost 
information (overwritten by the client render) or a double-render which can cause
some interesting (and unwanted) side-effects.

The best alternative that I've discovered so far is to set `id` as a required 
prop for these kinds of components, requiring the user to supply a valid value.
This is also less-than-desirable for reasons outlined in the Motivation section
above.

# Adoption strategy

This would be an opt-in feature and should only affect individual components 
that ask for a reserved ID. It should have no side-effects for components that 
don't opt in.  It should also not have any effect outside of the `hydrate` path.

# How we teach this

Since there is special behavior associated with these reserved IDs, care must be 
given when writing the documentation to distinguish this feature from other IDs
already present in HTML and React. Hence the suggestion of calling them 
`reservedIdentifier` to avoid confusion with the more ubiquitous `id`.

No re-organization needed for existing documentation. There would be a few 
additional entries to the documentation for `Component` as well as maybe a 
tutorial page.

# Unresolved questions

This RFC represents a high-level proposal for how the goal might be achieved. 
Due to my unfamiliarity with the internals of React, there are undoubtedly still 
many questions that would need to be answered. Some examples off the top of my 
head:

- Can the methodology described above for absorbing HTML attributes from the
  server-rendered markup be accomplished during the hydration phase before the 
  first DOM merge?

- Would this approach cover all use cases?

- Are there unintended and unwanted side-effects?

- Does this approach work with asynchronous rendering?
