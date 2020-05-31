- Authors: Michael Best (mbest)
- Start Date: 2020-06-01
- RFC PR: 
- React Issue: 

# Summary

Enhance React's reconciliation algorithm so that two different components will be considered equivalent if one component is marked as wrapping/extending the other component.

# Basic example

To indicate that a component wraps another component, set the former's `wraps` property to the latter component:

```js
IconButton.wraps = Button;
```
or

```js
class IconButton extends React.Component {
    static wraps = Button;
    ...
```

# Motivation

As mentioned in the [React docs](https://reactjs.org/docs/reconciliation.html#elements-of-different-types):

> Whenever the root elements have different types, React will tear down the old tree and build the new tree from scratch. Going from `<a>` to `<img>`, or from `<Article>` to `<Comment>`, or from `<Button>` to `<div>` - any of those will lead to a full rebuild.

[Elsewhere](https://reactjs.org/docs/higher-order-components.html#dont-use-hocs-inside-the-render-method):

> ... remounting a component causes the state of that component and all of its children to be lost.

In our project, we often create components whose purpose is to constrain or enhance another component. We've run into a problem with this approach when a component needs to change how it's displayed based on some state. 

For example, suppose we have an `IconButton` for a button that just contains an icon, and an `ActionButton` for one that shows text and an icon, and we want to switch a button between these two based on some state. Even though they both render a `Button`, React currently sees these as different components and re-creates the button itself when changing between them. If the button was in focus, it will lose focus from this state change.

# Detailed design

## Root components

The React reconciler will compare components based on their root component. For example, suppose `A` wraps `B` which wraps `X`, and `C` also wraps `X`, all of these component types will match. To facilitate faster comparison, React will add a `rootType` property to any component after following the `wraps` links. So each of `A`, `B`, `C`, and `X` will have `rootType` set to `X`. `React.createElement` will set this property if it's not already set.

## Reconciler

After comparing based on `type`, the reconciler will compare based on `rootType`. If these match, the reconciler will find the top-level matching component type and "detach" that existing element from the tree. Probably this can be done by marking that fiber as "used". Any existing non-matching elements will be "deleted". If the types now match directly, reconciliation continues as normal with the existing element getting the new props, etc. If they don't match, the existing detached element is used to compare with the new element's children, and so on. Consider the following scenarios using the example components above:

Current | New | Matches | Actions
--- | --- | --- | ---
`A>B>X` | `B>X` | `B` | Detach `B>X`; Delete `A`; Reuse `B>X`
`A>B>X` | `X` | `X` | Detach `X`; Delete `A>B`; Reuse `X`
`B>X` | `A>B>X` | `B` | Detach `B>X`; Add `A`; Reuse `B>X`
`X` | `A>B>X` | `X` | Detach `X`; Add `A>B`; Reuse `X`
`B>X` | `C>X` | `X` | Detach `X`; Delete `B`; Add `C`; Reuse `X`

## Keys

For elements to match they must have the same keys as well. So `<A key="wow">` will match `<B key="wow">` but not `<B key="hmm">`. The reconciler will set or clear the key of the existing elements based on the new structure.

## DEV warning

A component that defines a `wraps` property must render a single element of that type with no key, or return `null`. If it renders something else, React will output a warning in DEV mode.

## Interaction with higher-order components

HOCs that directly call the original component function should copy the `wraps` prop to the new component. Examples are `React.memo` and MobX's `observer`. HOCs that return a new component that renders the original component will likely want to set the `wraps` property on the new component.

# Drawbacks

## User space solutions

"Wrappers" can be done in user space using functions that return the base element. But this requires two methods of using components: JSX and functions.

## Has-a or is-a relationship

React components use composition, which generally represents the "has-a" relationship. A wrapper component more properly represent the "is-a" relationship, which explains the need to expose information about its "base" component. But perhaps this approach will be confusing, and a better system would more strictly expose and enforce the "inheritance" structure.

# Alternatives

A simpler approach would be to allow only prop-mapping when wrapping a component. Then `createElement` could perform the mapping and return a root-type element. But this means wrapper "components" wouldn't be able to use state, context, etc.

It might seem like ["reparenting"](https://github.com/reactjs/rfcs/pull/34) overlaps with this approach, but the wrapping method described here does not involve moving actual DOM nodes.

Possibly, React could also expose a `wrapper` function that sets the `wraps` property:

```js
const IconButton = React.wrapper(Button, (props) => {
    ...
    return <Button .../>;
});
```

# Adoption strategy

No developer will be required to use this technique. It will also be backwards compatible, since old React versions will just ignore the new prop.

# How we teach this

In the React docs, when examples of wrapped components are given, we should update them to include this API. For example:

- https://reactjs.org/docs/composition-vs-inheritance.html
- https://reactjs.org/docs/higher-order-components.html

The [reconciliation](https://reactjs.org/docs/reconciliation.html) docs should include a section about wrapped components.

The [API reference](https://reactjs.org/docs/react-component.html) should describe the new `wraps` prop along with the requirements for the component.

# Unresolved questions

 - Should we use the term "wraps"? The HOC docs already use "wrap" extensively and in ways that don't apply to this functionality. In the composition section, the term used is "specialization":
   > Sometimes we think about components as being “special cases” of other components.

- I've studied the React reconciler code enough to believe that this approach is possible, but not enough yet to feel confident writing the code. I'd probably need a bit of guidance. 
