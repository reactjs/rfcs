- Start Date: 2019-09-05
- RFC PR: (leave this empty
- React Issue: (leave this empty)

# Summary

Provide a shorthand syntax for passing variables as props, where the variable
name is same as the prop name; similar to the shorthand properties in ES6
object literals.

# Basic example

```jsx
function Toggle ({ onChange, checked, className }) {
    const toggleClasses = classd`checkbox-toggle ${checked && 'active'} ${className}`;

    function onClick () {
        onChange(!checked);
    }

    return <input type='checkbox' className={toggleClasses} +onClick +checked />;
}
```

This snippet transpiles to 

```jsx
function Toggle ({ onChange, checked, className }) {
    const toggleClasses = classd`checkbox-toggle ${checked && 'active'} ${className}`;

    function onClick () {
        onChange(!checked);
    }

    return <input 
        type='checkbox' 
        className={toggleClasses} 
        onClick={onClick} 
        checked={checked} />;
}
```

# Motivation

While using react, it happens quite a number of times that we need to:

i. pass a prop down the component chain with the same name
ii. pass a variable / function with the same name as the prop

In such cases, we need to write the same name twice as `propName={propName}`. This
is redundant. Also with the increase in the use of function components and
useState hooks, the usecase for such props is ever increasing.

Providing a shorthand syntax for passing such props would make the code
concise and encourage developers to write cleaner and less redundant code.

# Detailed design

The shorthand syntax, `+propName` (as recommended in this RFC) is to be supported
by existing JSX transpilers. It can be done by updating the parse and transform functions
for `JSXAttribute` nodes.

**NOTE.** This design is intended to work only with `JSXIdentifier` nodes prefixed with
`+` in the `JSXAttribute.name` field. It should not work with `JSXNamespacedName` and
should throw an `Unexpected Syntax Error` otherwise.

**In `parseJSXAttribute`**

1. check is the Token starts with a `+`, i.e. `charCodes.plusSign (ascii, 43)`
2. if true:
    i. eat the `tokens.plusMin` Token.
    ii. start a `JSXAttribute` Node `node`.
    iii. call `parseJSXIdentifier` and set `node.name` and `node.value` to the parsed `JSXIdentifier`.
    iv. finish and return `node`
3. otherwise continue with the existing `parseJSXAttribute` logic.

**In `transformJSXAttribute` visitor (entry phase)**

1. Check if the value in the current node (`path.node.value`) is a `JSXIdentifier`.
2. If true, set the value in the current node (`path.node.value`) to a `JSXExpressionContainer` containing an `Identifier` with the name same as the name of the `JSXIdentifier`. i.e. `types.jsxExpressionContainer(types.Identifier(path.node.value.name))` and return.
3. Otherwise continue with the existing visitor logic.


# Drawbacks

- It goes against the principle _"There should be one-- and preferably only one --obvious way of doing it"_. With the introduction of shorthand attributes, the way of writing JSX attributes becomes 4:
    1. The Standard `<Component propName={expr} />` 
    2. Spread attributes `<Component {...props } />` 
    3. Boolean attributes `<Component booleanProp />`
    4. Shorthand attributes `<Component +prop1 +prop2 />`
- There may be confusion with boolean attributes, the syntactic difference being only the `+` prefix.
- This proposes adding a new/different semantics to the existing unary `+` operator, even though this addional semantics is valid only within the context of `JSXAttributes` Nodes.

# Alternatives

- Maintaining the status quo, i.e. requiring the propName and the propValue to be written explicitly.
- Using any other symbol than `+` as the prefix, so that we do not add to semantics of existing operators. maybe `@`.

# Adoption strategy

This is not a breaking change and does not require any change to the codebase of React itself.
It needs to be implemented by JSX transpilers and can be made available with their release cycle.
Create-react-app, react-scripts, Linters and other existing tools need to support use of newer version of the
JSX transpiler (plugin). 

Existing react projects will still be valid with the newer syntax without requiring any change. But
styleguides and linters can provide the options like `--allow-jsx-shorthand-attributes` to process
the newer syntax as valid, and `--use-jsx-shorthand-attributes` to convert existing code to use 
the shorthand syntax where applicable.

# How we teach this

If accepted, the shorthand syntax can be specified and taught in the React/JSX documentation.
Conceptually using this shorthand syntax is optional, just like using `<> </>` for Fragments.
The teaching strategy, therefore, can be similar to that of the Fragments shorthand.
