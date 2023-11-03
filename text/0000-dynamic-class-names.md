- Start Date: 2023-04-18
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

The proposed feature is to add a new syntax for dynamically adding classes to React components. This syntax would be similar to the `ngClass` directive in Angular, allowing developers to use a `classNames` attribute to add or remove classes based on certain conditions.

# Basic example

```jsx
const [isHovering, isDisabled, isLoading] = [false, true, true];
const custom = 'custom';

<button classNames={[
  "primary",
  ["teal", "semi-transparent", isHovering && ["hover", "darken-2"]],
  { loading: isLoading, 'is-disabled': isDisabled },
  undefined,
  false,
  null,
  `${custom}-class`
]}>Click Me!</button>
```

**HTML Output:**
```html
<button class="primary teal semi-transparent loading is-disabled custom-class">Click Me!</button>
```

In this example, the `classNames` attribute is used to add several classes to the button component based on certain conditions. 

The button will have the following classes:
- primary
- teal
- semi-transparent
- hover (if `isHovering` is true)
- darken-2 (if `isHovering` is true)
- loading (if `loading` is true)
- is-disabled (if `isDisabled` is true)
- custom-class (normal string interpolation)

# Motivation

The motivation behind this proposed feature is to simplify class management in React and reduce the dependency on external libraries like `classnames`. In Angular, the `ngClass` directive allows for dynamic class management using a similar syntax, which has proven to be a popular and useful feature. By adding a similar syntax to React, developers can have a simpler and more streamlined way to handle class management, without needing to rely on external libraries. This can help reduce project complexity and make code easier to read and maintain.

# Detailed design

The proposed syntax for adding dynamic classes to React components would be through a `classNames` attribute, which would take an array as its value. Each element in the array would represent a class name, and could be either a string, an array of strings, or an object.

- A string element would be treated as a static class name and added to the component unconditionally.
- An array of strings would represent a set of nested class names. This would allow for complex class structures such as `teal` and `semi-transparent` along with `hover` and `darken-2` if `isHovering` variable is `true` without needing to use a separate library to generate them.
- An object element would allow for conditional class names, with the keys being the class names and the values being boolean expressions that determine whether the class should be added or removed.
In the example, the `isLoading` variable is used to conditionally add the `loading` class and `isDisabled` to add `is-disabled` class.

# Drawbacks

Why should we *not* do this? Please consider:

- The performance impact of using this syntax compared to other class management solutions.
- The potential for conflicts or edge cases with existing code (`className`) or CSS frameworks.
- The need for additional documentation and education to help developers learn the new syntax.

# Alternatives

The classnames library is currently the most common solution for dynamically managing classes in React. However, this requires an extra dependency and can add additional complexity to the project. Other alternatives include using inline styles or creating custom class management functions, but these approaches can also be cumbersome.

# Adoption strategy

The proposed syntax for dynamic classes could be introduced in a minor React release, along with documentation and examples to help developers migrate from other class management solutions. Additionally, tooling could be provided to help developers identify potential issues or conflicts

# How we teach this

*What names and terminology work best for these concepts and why? How is this idea best presented? As a continuation of existing React patterns?*

We recommend using the term "classNames" to refer to the proposed attribute for adding dynamic classes to React components. This term aligns with the existing "className" attribute in React and is consistent with the naming convention used in other programming languages and frameworks.

To teach this idea, we suggest presenting it as a continuation of existing React patterns. Specifically, we can demonstrate how the "classNames" attribute can be used in conjunction with the existing "className" attribute to provide a more flexible and powerful solution for adding classes to components.

*Would the acceptance of this proposal mean the React documentation must be re-organized or altered? Does it change how React is taught to new developers at any level?*

The acceptance of this proposal would require updating the React documentation to include information on the "classNames" attribute and how it can be used. This change would not significantly alter how React is taught to new developers, as it builds upon existing React patterns and conventions. However, it would provide a more convenient and intuitive solution for adding dynamic classes to components, which could improve the overall developer experience.

*How should this feature be taught to existing React developers?*

The acceptance of this proposal would require updating the React documentation to include information on the "classNames" attribute and how it can be used. This change would not significantly alter how React is taught to new developers, as it builds upon existing React patterns and conventions.

# Unresolved questions

- There may be a potential conflict between the existing "className" attribute in React and the proposed "classNames" attribute. It is unclear how this conflict should be handled - which attribute takes precedence, whether they should be combined, or whether one should overwrite the other and allow only one to be used. This issue needs further discussion and consideration to ensure that the proposed feature does not interfere with existing React functionality.