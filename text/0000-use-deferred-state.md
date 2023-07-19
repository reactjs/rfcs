- Start Date: 2023-07-19
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

This proposal suggests simplifying the usage of `startTransition` in `React` by
introducing a new hook.

# Basic example

Currently, in order to utilize `startTransition`, developers need to
import `useTransition` and set it up as follows:

```js
const [isPending, startTransition] = useTransition();
```

However, in some cases, the `isPending` value is not used, but it still needs to be included because `startTransition` occupies the second index of the `useTransition` output. Additionally, `startTransition` often needs to work in conjunction with `useState`. 

To address these complexities, I propose combining these hooks into a new one named `useDeferredState`

```js
const [text, setText, isPending] = useDeferredState();
```

Alternatively, we can achieve the same result using a HOF to wrap `useState`, like this:

```js
const [text, setText, isPending] = deferred(useState());
```

# Motivation
The main motivation behind this proposal is to streamline the usage of `useTransition` and `startTransition` in `React`, making it more straightforward and intuitive for developers. By introducing a new hook or a HOF, we can enhance the developer experience and promote cleaner, more concise code when working with transitions in React.

# Detailed design
`useDeferredState` and `deferred` are a combination of two main React hooks `useState` and `useTransition`.

By placing `isPending` as the third element in the output, it provides the flexibility to choose whether or not to utilize it in your code:
```js
// Use 'isPending' when needed:
const [text, setText, isPending] = useDeferredState();

// Omit 'isPending' when not required:
const [text, setText] = useDeferredState();
```

# Drawbacks

- I think teaching `useTransition` is much harder than teaching `useDeferredState`
- There is no breaking chaneg therefore there is no cost of migrating existing React applications.

# Alternatives

An alternative solution is having `useDeferredState` or `deferred` as
a 3th-party package.

```js
const useDeferredState = (initialValue) => {
    const [state, _setState] = useState(initialValue);
    const [isPenfing, startTransition] = useTransition();

    const setState = useCallback((newValue) => {
        startTransition(() => {
            _setState(newValue);
        });
    }, []);

    return [state, setState, isPenfing];
}

// usage:
const [text, setText, isPending] = useDeferredState();
```

or 

```js
const deferred = ([state, _setState]) => {
    const [isPenfing, startTransition] = useTransition();

    const setState = useCallback((newValue) => {
        startTransition(() => {
            _setState(newValue);
        });
    }, []);

    return [state, setState, isPenfing];
}

// usage:
const [text, setText, isPending] = deferred(useState());
```

# Adoption strategy

The proposed hook and Higher-Order Function are opt-in features, meaning that developers can
choose whether or not to use them in their code. Existing React applications that don't require
the new functionalities can continue using the traditional `useState` and `useTransition` approaches
without any changes.

# How we teach this

`useDeferredState`:
The proposed hook combines `useState` with a deferred state update mechanism to handle non-urgent updates. The term `deferred` accurately describes the behavior of the state update, as it delays execution until a later time (when React is less busy). This name aligns with React's existing hooks, such as `useEffect`, which also involve deferred operations.

`deferred`:
When introducing the Higher-Order Function, it wraps useState to achieve similar functionality to the `useDeferredState` hook. The name `deferred` directly reflects the primary purpose of the function, emphasizing its role in handling state updates in a deferred manner.

# Unresolved questions
Is there any downside if we use multiple `useTransition` in a component?