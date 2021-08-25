- Start Date: 2021-08-24
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Introduce a new hook to reduce/improve some process.

# Basic example

In this example we see `useEffect` that doesn't need a dependency to read updated `count`'s
value, so that's mean we don't need to perform `useEffect` effect function event `count` will
change. `useStateRef` just is a name, so might we need to change to better name.

```js
const [count, setCount, getCount] = useStateRef();

useEffect(() => {
  const handleVisibility = () => {
    console.log(getCount());
  };

  document.addEventListener('visibilitychange', handleVisibility);

  return () =>  document.removeEventListener('visibilitychange', handleVisibility);
}, []);
```

Also, in this example `useCallback` doesn't need a dependency to know `count`'s value, so
`useCallback` inside function just perform once. that's awesome

```js
const [count, setCount, getCount] = useStateRef();

const handleClick = useCallback(() => {
  console.log(getCount());
}, []);
```

# Motivation

`useEffect` and `useCallback` needs to know dependencies to redefine `function` depended
on new values of dependencies. but these are redundant process for `React` because we don't
know what time we need it exactly ex: `handleClick` depend on user behavior so we just need
`count`'s value when user need it.

# Detailed design

A basic of implementation for `useStateRef` that we can use in project is:
```js
function useStateRef<T>(initialValue: T): [T, (nextState: T) => void, () => T] {
  const [state, setState] = useState(initialValue);
  const stateRef = useRef(state);
  stateRef.current = state;
  const getState = useCallback(() => stateRef.current, []);
  return [state, setState, getState];
}
```

# Drawbacks

This is a new hook, so we don't have any breaking change, also we can implement that by
internal React hooks.

# Alternatives

Alternative can be a package, maybe

# Adoption strategy

Fortunately we don't have any breaking change, also we can embed this hook to `useState` without
breaking change

```js
const [count, setCount, getCount] = useState();
```

# How we teach this

This RFC introduce a new hook, so we have to add some section to React documentation

# Unresolved questions

- do we need to implement a new hook or embed it to `useState`?