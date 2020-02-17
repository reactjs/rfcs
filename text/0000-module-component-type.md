- Start Date: 2020-02-15
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

If `MyComponent` is an ES Module, then `<MyComponent>` should behave the same as `<MyComponent.default>`

(e.g. transpile to `React.createElement(MyComponent.default)`).

# Basic example

In the following code:

```jsx
import * as Tabs from './tabs';

export default = () => <Tabs>
    <Tabs.Tab />
</Tabs>
```

Resolve `<Tabs>` as `<Tabs.default>`.

# Motivation

A [common](https://kentcdodds.com/blog/compound-components-with-react-hooks) way of exporting interlinked React Components is to use a [Compound Component pattern](https://egghead.io/lessons/react-write-compound-components):

```jsx
const Tabs = ({children}) => <div>
    { children }
</div>;

const Tab = ({children}) => <div>
    { children }
</div>;

Tabs.Tab = Tab;

export default Tab;
```

This results in a syntax that clearly shows the relationship between the two components:

```jsx
import Tabs from './tabs';

export default = () => <Tabs>
    <Tabs.Tab />
    <Tabs.Tab />
</Tabs>;
```

Unfortunately, if a compound component contains several optional child components, this technique does not benefit from tree shaking.

This can be resolved as follows:

```jsx

const Tabs = ({children}) => <div>
    { children }
</div>

const Tab = ({children}) => <div>
    { children }
</div>;

export { Tabs, Tab };
export default Tab;
```

But the code utilising this code must look as follows:

```jsx
import * as Tabs from './tabs';

export default = () => <Tabs.Tabs>
    <Tabs.Tab />
    <Tabs.Tab />
</Tabs.Tabs>;
```

or

```jsx
import * as Tabs from './tabs';

export default = () => <Tabs.default>
    <Tabs.Tab />
    <Tabs.Tab />
</Tabs.default>;
```

The addition of `.Tabs`/`.default` complicates the code, especially when dealing with larger component names.

On the other hand, having `<Tabs>` resolve to `<Tabs.default>` would be a cleaner syntax, but this would require a change to JSX transpilation or React runtime.

# Detailed design

I see 2 possible routes, though am still assessing them:

## Change to babel-plugin-react-jsx

If we can statically analyse `MyComponent` and identify it is an es module, then it may be possible to do this during transpilation.

Need to consider whether we could always assess this during transpilation, or whether a 'works in some cases' approach is acceptable.

e.g. if someone passes a Module through to a higher order component.

## Handle at runtime

e.g. in `packages/react-reconciler/src/ReactFiber.js`, after case statements but before error:

```js
          if (type.default != null) {
            return (createFiberFromTypeAndProps(type.default,
              key, pendingProps, owner, mode, expirationTime))
          }
```

TBC what method could be used to reliably detect an ES Module, that still works after transpilation to a commonjs Module.

# Drawbacks

- Handling at runtime may cause issues in transpiled code, when trying to detect whether a component passed in was originally an esModule.
- I don't know enough about the babel plugin to identify issues

# Alternatives

As the main motivation is to be able to use clean syntax for Compound Components and still get the benefits of tree shaking, an alternative may be to find another way of defining a compound component, or to make improvements to tree shaking.

# Adoption strategy

- add a Compound Components section to 'Advanced Guides' in React documentation.
- not a breaking change (tbc, depends on implementation, may be issues with HOCs or external functions checking if an object is a valid React component, that would reject a Module)

# How we teach this

*What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing React patterns?*
*How should this feature be taught to existing React developers?*

Add a Compound Components section to 'Advanced Guides' in React documentation, that covers the existing premise plus tree shaking considerations.

*Would the acceptance of this proposal mean the React documentation must be
re-organized or altered?*
*Does it change how React is taught to new developers at any level?*

No


# Unresolved questions

- Whether to execute during transpilation or at runtime.
- If this is done at runtime, and the entire Module is passed to `React.createElement`, would tree shaking still be able to remove unused exports?
