- Start Date: 2018-01-10
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Add React.Children.isRenderable() method which returns `boolean` depending
on whether a passed child returns `ReactNode` or not.

# Basic example

`React.Children.isRenderable` lets us easily detect children without UI.
```js
const MaybeWrapper = ({children}) =>
    React.Children.map(children, child =>
        React.Children.isRenderable(child)
            ? <Wrapper>{child}</Wrapper>
            : child
    )
```

# Motivation

Given a component which renders markup or `null` depending on its props there
are cases when we want to know whether it renders something:
- if we need to wrap it with layout, or draw something else around or add
animations, etc and don't want to break styling with excessive wrappers;
- if we want to show a placeholder when the component has not been rendered;
- if we want to count actually visible children in an array.

`React.Children.isRenderable` would allow us easily check if a component
renders anything without messing up it's internals. It is simple, useful and
plays well with other methods from `React.Children` namespace.

# Detailed design

A method takes a `child` component instance, finds it's corresponding fiber and
checks whether it has a `stateNode`. If a `stateNode` exist, the method returns
`true`, else it starts recursively repeat this process on a child fiber and it's
siblings until it finds a non-empty `stateNode` containing a DOM node, otherwise
it returns `false`.

# Drawbacks

Why should we *not* do this? Please consider:

- the proposed feature can be implemented in user space

# Alternatives

Obviously, we could store the result of the rendering condition in the
component's state, update it when props change and check passing handler from
a parent or we could hoist condition to a parent but we might not want do
anything of that because:
- the component may receive props from context (e.g. from Redux store via
`connect` function) so we cannot hoist the condition;
- adding props for "renderability" handler to the component may break coherence;
- we may not be able to modify the component's internals (e.g. it is from a 3rd
party library);
- we don't want to add complexity for a trivial action (e.g. we have
a stateless functional component with rendering condition. Do we really need
to make it stateful just to check if it renders anything?) or lose
convenience (stateless conditional components are quite eloquent).
We could also use refs to check if a component was rendered, but this is even
more messy solution than previous.

# How we teach this

To understand why this feature is useful and how it should be used, a user
must be familiar with core React concepts covered in [React Components, Elements, and Instances](https://reactjs.org/blog/2015/12/18/react-components-elements-and-instances.html)

# Unresolved questions

Another possible solution might be to create a method, which returns a complete
Node (or `null`), rendered by the passed child though such functionality may
lead to misuse and confusion. Also that would probably have some performance
drawbacks because it requires to traverse deeper into a fiber.

The name of this method might be reconsidered. The alternatives are:
`hasNode`, `hasReactNode`, `rendersNull` (then should return `false` if
renders anything).
