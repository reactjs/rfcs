- Start Date: 2025-06-26
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

React has a special prop called `key` used to distinguish between rendered instances of a component within the same parent. It's essential for mapping a data array to a component array.

The `key` prop also has an application outside of mapped components. Parents can use it to force a component to re-render by changing the `key` prop value. This is useful when you want to reset the state of a component or trigger a re-render without changing the component's props.

This technique is described in [You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect#resetting-all-state-when-a-prop-changes).

However, writing code where the parent is responsible for the state reset is somewhat of a "tail wagging the dog" situation, as often the component itself is the one that knows best when it needs to reset its state. This RFC proposes allowing components to use the `key` prop behavior in a more flexibly, enabling them to control their own re-rendering behavior by introducing a new hook called `useKey`.

# Basic example

From the [You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect#resetting-all-state-when-a-prop-changes) article:

Problematic component:

```jsx
export default function ProfilePage({ userId }) {
  const [comment, setComment] = useState('');

  // 🔴 Avoid: Resetting state on prop change in an Effect
  useEffect(() => {
    setComment('');
  }, [userId]);
  // ...
}
```

Current solution (note: the parent is responsible for resetting the state):

```jsx
export default function ProfilePage({ userId }) {
  return (
    <Profile
      userId={userId}
      key={userId}
    />
  );
}

function Profile({ userId }) {
  // ✅ This and any other state below will reset on key change automatically
  const [comment, setComment] = useState('');
  // ...
}
```

Proposed solution (note: the component is responsible for resetting its own state):

```jsx
export default function ProfilePage({ userId }) {
  // ✅ This and any other state below will reset on key change automatically
  useKey(userId);
  const [comment, setComment] = useState('');
}
```

# Motivation

A component should be able to control its own re-rendering behavior, especially when it comes to resetting its state. The current pattern of using the `key` prop requires the parent component to manage this behavior. This leads to the same components' internal state behaving differently depending on whether the parent has passed a `key` prop or not, which can be confusing and error-prone.

A component should be able to identify whether it is a new instance of itself or not. This will help developers of higher-level components avoid thinking about the inner workings of the lower-level components.

# Detailed design

The idea is to implement a new hook called `useKey` that can be used within a component to indicate that the component should reset its state when the value passed to `useKey` changes.

The signature of the hook would be:

```jsx
void useKey(value: React.Key);
```

> Note that, unlike the `key` prop, `useKey` does not accept `null` or `undefined`. This is because, in compound key situations, `useKey` would serialize both values to the same representation as an empty string, which could lead to surprising behavior. It is preferred for the caller to coalesce null or undefined values to a string representation before passing them to `useKey`.

The `useKey` hook would work similarly to the `key` prop, but it would be used within the component itself. When the value passed to `useKey` changes, the component will be treated as a new instance, and its state will be reset.

`useKey` would be able to combine itself with the `key` prop, allowing for more complex scenarios where both the parent and the component can control the re-rendering behavior by generating a compound key.

For instance, if a component uses `useKey("bar")` and the parent passes `key="foo"`, the component's effective key would be `"foo-bar"`.

Likewise, multiple `useKey` calls within the same component would compound together, allowing for more complex scenarios where multiple keys are needed to determine the component's identity.

```tsx
export default function ProfilePage({ userId, organizationId, projectId }) {
  // ✅ Equivalent to the parent passing a key prop with `${userId}-${organizationId}-${projectId}`
  useKey(userId);
  useKey(organizationId);
  useKey(projectId);
  const [comment, setComment] = useState('');
  // ...
}
```

Given that React's reconciliation algorithm expects to have a determined `key` _before_ processing the body of a component, the `useKey` hook would need to be called at the top level of the component, before any other hooks. This would ensure that the component's identity is established before any state or effects are initialized.

This is particularly important to avoid `useKey` consuming dynamic values that could change during the component's lifecycle such as state or suspense hooks.

```tsx
export default function ProfilePage({ userId }) {
  const [comment, setComment] = useState('');
   // 🔴 Bad: call `useKey` with dynamic internal state is inappropriate
  useKey(projectId);
  // ...
}
```

More broadly, calls to `useKey` should be strictly derived from props. 

Aside from the requirement of being called at the top level of the component, `useKey` would behave similarly to other hooks and force rules of hooks to be followed. This means it cannot be called conditionally or within loops.

# Drawbacks

- `useKey` is technically possible to call within other hooks. This could lead to library hooks hijacking the behavior of components and unintentionally resetting all state.
- This seems like it would make the rendering process more complex, as React would need to track both the `key` prop and the `useKey` hook within the same component. This might be challenging given that the `useKey` portion would not be known until the component is rendered, while the `key` prop is known at the time of reconciliation.
- It could lead to confusion for developers who are not familiar with the `useKey` hook, as it introduces a new way of thinking about component re-rendering and state management.
- It could be used in scenarios where it is not necessary, leading to unnecessary re-renders and performance issues.
- This is already possible with Higher Order Components or purpose-built wrapper components.
- Breaks existing assumptions about hooks since it may not be called in any order with other hooks, but rather at the top level of the component.

# Alternatives

The impact of not implementating the RFC would be that developers would continue to use the `key` prop in the parent component to reset the state of child components. This would lead to the same issues described in the motivation section, where the parent is responsible for managing the child's state reset behavior. In other words, the tail will keep wagging the dog.

An alternative to this RFC, which can already be implemented in userland are HOCs or wrapper components that manage the `key` prop behavior. They can look like this:

```jsx
function ProfilePage({ userId, organizationId, projectId }) {
  const [comment, setComment] = useState('');
  // ...
}

export default withKeys(ProfilePage, (props, Component) => {
  return <Component key={`${props.userId}-${props.organizationId}-${props.projectId}`} {...props} />;
});
```

I don't like this approach because:

1. It requires additional boilerplate code to create the HOC or wrapper component.
2. There are a dozen subtly different ways to implement this behavior, which can lead to inconsistencies across codebases.
3. It does not resolve the fundemental issue that the component itself should be responsible for managing its own re-rendering behavior.

# Adoption strategy

It is not a breaking change for most applications. Some libraries that lean on APIs like [`cloneElement`](https://react.dev/reference/react/cloneElement) and may encounter unexpected behaviors if they attempt to own the `key` prop themselves.

Such libraries will need to identify if the `useKey` hook is a breaking change for them. Those cases should be rare.

# How we teach this

This should be taught as part of the [You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect#resetting-all-state-when-a-prop-changes) article, along a new page dedicated to state resetting.

This hook recycles the same principles as the `key` prop, so it should be easy to understand for developers who are already familiar with React.

# Unresolved questions

- Is this possible to implement within the reconciliation algorithm?
- In what cases might this be a breaking change?
- Are there any cases where not calling `useKey` at the top level of the component would be appropriate?
- Should `useKey` be preferred to the `key` prop in all cases, or are there scenarios where the `key` prop is still the better option?
