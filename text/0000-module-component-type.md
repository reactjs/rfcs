- Start Date: 2020-02-15
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

If `MyComponent` is an esModule, then resolve `<MyComponent>` as `React.createElement(MyComponent.default)`.

# Basic example

In the following code:

```jsx
import * as Tabs from './tabs';

export default = () => <Tabs>
    <Tabs.Tab />
</Tabs>
```

Resolve `<Tabs>` as `React.createElement(Tabs.default)`

# Motivation

A common way of exporting interlinked React Components is to use a Compound Component pattern:

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

The addition of `.Tabs` complicates the code, especially when dealing with larger component names.

On the other hand, `<Tabs>` should suffice, but this would require a change to JSX transpilation or React runtime.

# Detailed design

I see 2 possible routes, though am still assessing them:

## Change to babel-plugin-react-jsx

If we can statically analyse `MyComponent` and identify it is an es module, then it may be possible to do this during transpilation.

## Handle at runtime

e.g. in `packages/react-reconciler/src/ReactFiber.js`, after case statements but before error:

```js
          if (type.default != null) {
            return (createFiberFromTypeAndProps(type.default,
              key, pendingProps, owner, mode, expirationTime))
          }
```

# Drawbacks

- Handling at runtime may cause issues in transpiled code, when trying to detect whether a component passed in was originally an esModule.
- I don't know enough about the babel plugin to identify issues

# Alternatives

What other designs have been considered? What is the impact of not doing this?

# Adoption strategy

If we implement this proposal, how will existing React developers adopt it? Is
this a breaking change? Can we write a codemod? Should we coordinate with
other projects or libraries?

# How we teach this

What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing React patterns?

Would the acceptance of this proposal mean the React documentation must be
re-organized or altered? Does it change how React is taught to new developers
at any level?

How should this feature be taught to existing React developers?

# Unresolved questions

Optional, but suggested for first drafts. What parts of the design are still
TBD?