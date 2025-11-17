- Start Date: (2024-10-6)
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

React at first had class components. Creating a class component was hard. So it introduced function components.
Function components were easier to create. They were more readable. They had less boilerplate.
Then server components came into the scene.

Now I would like to propose *light components*. They are simple valid JSX variables or constants. They do not have props and they do not have states. They are just a simple JSX bundle.

# Basic example

```
const Contacts = <div>
   <div>some phone number</div>
   <div>some email</div>
</div>

<Contacts />
```

# Motivation

It's all about unnecessary boilerplate. Our team has built an SPL (software product line). The panel is built using React.
We minimize the boilerplate as much as we can. The reduced boilerplate adds direct business value to us (enhanced maintainability, lower learning curve, faster time-to-market).

One aspect of the boilerplate is to be forced to add at least two parentheses and a lambda (4 characters) to turn a JSX constant into a function component:

```
const Contacts = () => <div>
   <div>some phone number</div>
   <div>some email</div>
</div>

<Contacts />
```

It does not look much in one single instance. But multiply it by more than 500 lists that we have, and each list has a couple of JSX constants and the value shows itself.

For example, look at this `/SomePath/Product/List.jsx` file:

```
const headers = <>
    <th>Name</th>
    <th>Price</th>
</>

const row = product => <>
    <td>{product.name}</td>
    <td>{product.price}</td>
</>

const listActions = <>
    <Categories />
    <Tags />
    <Attributes />
    <Export />
</>

const entityActiions = <>
    <Images />
    <Categories />
    <Tags />
    <Price />
</>

const Products = <List
    title='Products'
    entityType='Product'
    headers={headers}
    row={row}
    listActions={listActions}
    entityActions={entityActions}
/>

export default Products
```

Rendering a JSX fragment inside a file/component is not hard. You render it like `{jsxFragment}`. But when it comes to big systems where you split the system into parts, and develop part separately and reuse parts, that's when it becomes hard.

# Detailed design

I recommend that we start with a build configuration flag. In CRA (WebPack) or Vite or any other build system we can configure React to treat JSX framgnets/constants as components:

```
{
    treatJsxAsComponent: true
}
```

Then we should be able to capitalize the first letter of the JSX constant/variable and render it like a normal component:

```
const LightComponent = <div>Light component without props and state</div>

<LightComponent />
```

The point is that light components won't accept props (they can ignore them with a warning in the console) and they won't have states.

```
<LightComponent someKey="someValue" />
// creates a warning in the console that light components won't support props and those props will be ignored
```

# Drawbacks

Why should we *not* do this? Please consider:

Since we can create this behavior as a progressive configurational behavior, I can think of no drawback. Any team who wants light components can turn on a flag and get a new behavior. Otherwise, React is just like how it was before.

# Alternatives

There are a couple of alternatives. One is to use something like a higher-order component to get the JSX constant, and render it.

```
const Contacts = <div>Some contact data here</div>

const JsxRenderer = ({ jsx }) => {
    return <div>
        {jsx}
    </div>
}

<JsxRenderer jsx={Contacts} />
```

But that adds to the boilerplate even more. It defies the first goal which is to reduce the boilerplate.

# Adoption strategy

This feature won't be a breaking change. At first, it can be enabled via a flag. That way no team would be affected by it.
It can stay as a flag for the rest of the React lifecycle.
However, if it become a permanent behavior, then no team should be affected. Because both `<LightComponent />` and `{LightComponent}` should be OK and work.

# How we teach this

A new section should be created in the docs called `Light components`. It should explain some limitations for these types of components.
It should also present some examples on when are they useful. And example could be a `<SalesBadge />` simple JSX that can be created in one place and be reused across an online shop without any change.
