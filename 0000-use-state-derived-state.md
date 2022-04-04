- Start Date: 2020-03-04
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Add a second function argument to `useState` to replicate `getDerivedStateFromProps`.

# Basic example

```js
const Example = ({ items }) => {
  const [selectedIndex, setSelectedIndex] = React.useState<number | undefined>(
    undefined,
    s => s !== undefined && s >= 0 && s < items.length ? s : undefined
  )
  
  // Other component code
}
```

# Motivation

It's currently possible to do this via a `setState` call within the render method. However, this is has some downsides.

* Your logic must not call `setState` if the state does not change to avoid infinite loops - and the errors aren't always so straight forward to follow
* It's easy to accidentally run the first render with invalid values - which may not be an issue if it's only used in `useEffect` - however, later code changes can cause crashes (i.e. when your state is an index to some other data structure)
* You cause a second render pass

Having a more ergonomic API for this somewhat common usecase will lead to fewer bugs.

# Detailed design

The second function is always evaluated at component render time, and is called with the current state.

If it returned value is the same as the state passed in, it behaves as a normal current-day `useState` hook.

If the returned value differs from the state passed in, the state is internally updated, and the tuple returned has the `0th` element set to the new value. No additional render pass is executed on the component.

In development mode, whenever the function returns a new value, the function will be executed again with the new value passed in, and an error is thrown if the value changes an additional time.

The eslint plugin can remain the same.

# Drawbacks

* Additional APIs
* Makes `useState` and `useReducer` less unified (assuming we don't want to add this feature to `useReducer`)
* This can be implemented in user-space, at the cost of the lint plugin being overly conservative about dependencies
* Creates two approaches to update state at render time, rather than just the one we have today

# Alternatives

```js
const useStateConstraint = <S>(
  initialState: S | (() => S),
  getDerivedState: (state: S) => S,
): [S, Dispatch<SetStateAction<S>>] => {
  const stateTuple = useState(initialState);
  const prevState = stateTuple[0];
  const nextState = getDerivedState(prevState);
  if (!Object.is(prevState, nextState)) {
    if (__DEV__) {
      if (getDerivedState(nextState) !== nextState) {
        throw new Error('Expected useStateConstraint to return stable value');
      }
    }

    const setState = stateTuple[1];
    setState(nextState);
    return [nextState, setState];
  } else {
    return stateTuple;
  }
}
```

# Adoption strategy

Not a breaking change.

# How we teach this

This is already taught in `getDerivedStateFromProps` - and there's [documentation for the current-day hooks equivalent](https://reactjs.org/docs/hooks-faq.html#how-do-i-implement-getderivedstatefromprops).

# Unresolved questions

Nil.
