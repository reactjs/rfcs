- Start Date: 2024-11-18
- RFC PR: https://github.com/reactjs/rfcs/pull/261
- React Issue: (leave this empty)

# Summary

This RFC proposes to automatically apply `useIsolation(() => …)` (see https://github.com/reactjs/rfcs/pull/257) to all custom hooks using the [React compiler](https://react.dev/learn/react-compiler).

# Basic example

Let’s say someone write this custom hook in their code:

```jsx
function App() {
  const status = useLongPoll('/url', 100);
}

// ...

function useLongPoll(url, delay) {
  const [id, setId] = React.useState(0);

  React.useEffect(() => {
    const intervalId = setInterval(() => setId((id) => id + 1), delay);
    return () => clearInterval(intervalId);
  }, [delay]);

  const [status, setStatus] = React.useState();
  React.useEffect(() => {
    const controller = new AbortController();
    fetchStatus(url, { signal: controller.signal }).then((result) =>
      setStatus(result)
    );
    return () => controller.abort();
    // Trigger a re-fetch every so often
  }, [id]);

  return status;
}
```

The React compiler could automatically _patch_ it to look like the following:

```jsx
function App() {
  const status = useIsolation(() => useLongPoll('/url', 100), []);
}

// ...

function useLongPoll(url, delay) {
  const [id, setId] = React.useState(0);

  React.useEffect(() => {
    const intervalId = setInterval(() => setId((id) => id + 1), delay);
    return () => clearInterval(intervalId);
  }, [delay]);

  const [status, setStatus] = React.useState();
  React.useEffect(() => {
    const controller = new AbortController();
    fetchStatus(url, { signal: controller.signal }).then((result) =>
      setStatus(result)
    );
    return () => controller.abort();
    // Trigger a re-fetch every so often
  }, [id]);

  return status;
}
```

# Motivation

This topic is mostly for performance reasons, see [comment in the original RFC](https://github.com/Ayc0/react-rfcs/blob/main/text/0000-use-isolation.md#wrapping-existing-hooks-for-perf-optimizations-only).

One of the main problems when creating custom hooks / calling custom from 3rd party libraries, is that they can depend on their own update cycles, and they can create a lot of renders that may not be used in the end to drive UI changes. `useIsolation` introduced in another RFC can fix that, but developers still have to manually apply it when needed.

With the introduction of the React compiler, the compiler automatically applies the equivalent of `useMemo`, it seems fitting to also extend it to `useIsolation` (also as the React team planned to potentially extend `useMemo` to work exactly like this `useIsolation`, it was previously mentioned on Twitter, but I have no link to add here as those tweets have been deleted since).

# Detailed design

We can use the same rule used in the React compiler that automatically wraps variables in `useMemo`, to detect when custom hooks are getting called (check if their names match `/use[A-Z].*/`), and automatically wraps them in `useIsolation`. We can also use the same rule to check which variables to add in the dependencies array, or fallback to `[]` if no external variable is used.

> [!note]
> As mentioned in https://github.com/reactjs/rfcs/pull/257, the dependency array is not required for `useIsolation` to work. But this will allow fully skipping the call to the hook if nothing changed, skipping a few checks (one for each hook used in those, no need to perform `useState` resolution, or cache checks for `useMemo` / `useCallback`, etc.).
> So this would still be a win: 1 cache check to avoid a lot

# Drawbacks

If the written code properly follows the rule of hooks, no code should depend on render cycles. So automatically applying `useIsolation()` should be fine.
For the code paths not strictly following them, we can follow the current behavior of the compiler: not doing anything.

Also, this would add 1 `useIsolation` call in all components, adding an extra cache check for each component, which can make the memory usage higher, and potentially slow down (at least on initial mounts, and/or on non-properly memoized architecture).

# Alternatives

## Wrap at definition

Instead of wrapping the call sites like proposed, potentially we could wrap the hooks at their definitions, like so:

```jsx
function App() {
  const status = useLongPoll('/url', 100);
}

// ...

function useLongPoll(url, delay) {
  const [id, setId] = React.useState(0);

  React.useEffect(() => {
    const intervalId = setInterval(() => setId((id) => id + 1), delay);
    return () => clearInterval(intervalId);
  }, [delay]);

  const [status, setStatus] = React.useState();
  React.useEffect(() => {
    const controller = new AbortController();
    fetchStatus(url, { signal: controller.signal }).then((result) =>
      setStatus(result)
    );
    return () => controller.abort();
    // Trigger a re-fetch every so often
  }, [id]);

  return status;
}
```

becoming

```jsx
function App() {
  const status = useLongPoll('/url', 100);
}

// ...

function useLongPoll(url, delay) {
  return useIsolation(() => _useLongPoll(url, delay), [url, delay]);
}

function _useLongPoll(url, delay) {
  const [id, setId] = React.useState(0);

  React.useEffect(() => {
    const intervalId = setInterval(() => setId((id) => id + 1), delay);
    return () => clearInterval(intervalId);
  }, [delay]);

  const [status, setStatus] = React.useState();
  React.useEffect(() => {
    const controller = new AbortController();
    fetchStatus(url, { signal: controller.signal }).then((result) =>
      setStatus(result)
    );
    return () => controller.abort();
    // Trigger a re-fetch every so often
  }, [id]);

  return status;
}
```

But this has a few downsides:
1. It changes the name of the hooks, so for error handling and similar, it can add some additional cognitive costs for devs,
2. Can we parse 3rd party code? When wrapping hooks used in your app, you can automatically wrap all the used ones, so it's _fairly easy_. But detecting all the defined hooks can be more complicated (specifically as someone could name a regular function `useHello` without it being a hook),
3. In this example, we can see that we created the dependency array `[url, delay]`. But with the original example, it created `[]` (because the variables were fully static.) So this creates more memory load as we aren't tracking exactly how those hooks are used, but how they are defined.

## Wrap as `useMemo`

I lied a bit about the React compiler: it does **not** add `useMemo`. Instead the following code:

```jsx
function MyApp() {
  const [state] = React.useState({})

  const copied = React.useMemo(() => JSON.stringify(state), [state])

  return <span>{copied}</span>
}
```

will be transformed into ([see playground](https://playground.react.dev/#N4Igzg9grgTgxgUxALhAMygOzgFwJYSYAEAsgJ4CCADlQBQCURwAOsUXIWDkQNpcCGOBAF0iAXiIAlBP1wA6KGAQBlHIIS1gAX3qtWRdp24cqeBABNxUmfMUISCALYRaDcQD4iAKWUB5AHJyXDB4mADmeGhktAJC9AA0vLEiupj6RDAIOLDEADxgVPyY7sAmZuZauQD0BUXurFqsIFpAA)):

```jsx
function MyApp() {
  const $ = _c(5);
  let t0;
  if ($[0] === Symbol.for("react.memo_cache_sentinel")) {
    t0 = {};
    $[0] = t0;
  } else {
    t0 = $[0];
  }
  const [state] = React.useState(t0);
  let t1;
  let t2;
  if ($[1] !== state) {
    t2 = JSON.stringify(state);
    $[1] = state;
    $[2] = t2;
  } else {
    t2 = $[2];
  }
  t1 = t2;
  const copied = t1;
  let t3;
  if ($[3] !== copied) {
    t3 = <span>{copied}</span>;
    $[3] = copied;
    $[4] = t3;
  } else {
    t3 = $[4];
  }
  return t3;
}
```

And this piece of code is the equivalent of `const copied = React.useMemo(() => JSON.stringify(state), [state])`:

```jsx
  let t1;
  let t2;
  if ($[1] !== state) {
    t2 = JSON.stringify(state);
    $[1] = state;
    $[2] = t2;
  } else {
    t2 = $[2];
  }
  t1 = t2;
  const copied = t1;
```

Potentially, if the React team ships the fact that `useMemo` can be used to wrap hooks too, we could use the same syntax (with the tags `$[n]` and variables `tn`) for this RFC too.

I don't think it's currently possible (with the current implementation of the compiler, I mean), because in https://github.com/reactjs/rfcs/pull/257, I'm mentioning:

> 2. Create a new internal _call scope_ (like a component)

And to the best of my knowledge, using those `if`s can't let you create those _call scope_. Maybe the compiler can be updated to allow defining them (I have no idea how, I don't have the knowledge about how React would define them / expose them.)
But if we just look at the current compiler output, I believe that addings those `$[n]` would be harder that keeping in the compiled code calls to `useMemo` **for hooks** optimizations.

# Adoption strategy

I have no idea.

# How we teach this

As the compiler is a bit magical, I don’t if we need to teach that in a specific way. And if we do, we can mention something like "automatically wrap custom hooks in `useIsolation()`

# Unresolved questions

- Do we even want to apply the RFC https://github.com/reactjs/rfcs/pull/257?
- Is it something that the compiler can do?
- Is it the best way of automatically applying `useIsolation`?
- If the React team prefers the `useMemo` approach, does it fit how the compiler already handles `useMemo`?
- Is it really worth it?
