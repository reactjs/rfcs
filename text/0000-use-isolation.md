- Start Date: 2023-12-13
- RFC PR: https://github.com/reactjs/rfcs/pull/257
- React Issue: (leave this empty)

# Summary

This RFC proposes a new hook `useIsolation(() => …)` to allow creating a _sub-hook_ that can compute expensive values in _isolation_ and can memoize its returned value.

A simple equivalent would be to introduce a new hook that would behave like `useMemo`, but in which you can also use other hooks.

# Basic example

Let’s say you have a context that is a bit massive, but you just want to listen to 1 value `foo` in it and only re-render when it changes, you could do something like this:

```jsx
const MyComponent = () => {
  const foo = React.useIsolation(() => {
    const context = React.useContext(CustomHeavyContext);
    return context.foo;
  });
};
```

But this would also work with any kind of state / values:

```jsx
const MyComponent = () => {
  const bar = React.useIsolation(() => {
    const allSearchParams = useSearchParams();
    return allSearchParams.get("bar");
  });
};
```

# Motivation

This topic is mostly for performance reasons. A few of other RFCs are proposing solutions to solve this:

- `useIsolation` in https://github.com/reactjs/rfcs/issues/168,
- `useContextSelector` in https://github.com/reactjs/rfcs/pull/119.

When working with components, you can bail out of re-renders with `React.memo` / `shouldComponentUpdate()`.

And given this tree: `<A><B/></A>`, when `A` changes, `B` may not re-render.<br>
But when working with hooks, if you have 2 hooks `useA` and `useB` (used within `useA`), when `useB` re-renders, there is no way to bail out from its renders in `useA`.

> [!NOTE]
> The goal of this RFC is not to provide a way for `useB` to not re-render when `useA` re-renders (even if it’s mentioned a bit with dependencies later), but to **not always** have to re-render `useA` when `useB` re-renders.

This is an issue for large codebases that can share a lot of hooks (without having the ability of auditing all of them).<br>
Or for libraries like `react-router-dom` that expose large objects (the router state) and where users want to only register for changes that matter to them.


# Detailed design

This hook is inspired by both `React.memo` where you can provide a `areEqual` function as a 2nd parameter, and by `useMemo`.

When you want to make a piece of code run in _isolation\*_ from its parent component/hook, you can use `useIsolation`.<br>
This hook takes 2 parameters:

- Using MDN’s notation: `useIsolation(callback, [deps])`
- Using TS’s notation: `useIsolation<T>(callback: () => T, deps?: ReadonlyArray<unknown>)`

\*: this wouldn’t strictly run in isolation, as you may define dependencies to ease the communication between the parent scope and the isolated one.

## Pseudo-code mechanism

The way this would work in pseudo-code is this way:

> [!WARNING]
> I don’t know React internals so I’m going to make some assumptions

1. During the initial mount, call the `callback` within its _parent scope_
2. Create a new internal _call scope_ (like a component)
3. If the `callback` uses hooks like `useState`, `useContext`, etc., bind them to this _call scope_ instead of its parent scope
4. If there are dependencies defined, store then in _internal slots_ (like with `useMemo` or `useCallback`) saved on the _parent scope_
5. Store the return value of the `callback` in a _internal slot_ in the _parent scope_
6. If there are any updates in any of the hooks defined within the _call scope_ (aka within the `callback`), re-compute the `callback` within its parent scope (just like when `useMemo` recomputes), and compare the return value in previous _internal slot_, if it’s the same, do nothing, if it changes, update the _internal slot_ and re-render the _parent scope_.
7. If there are any updates in the _parent scope_, check if any dependencies have changed (if no dependencies are set, recompute on all updates), re-compute the `callback` within its parent scope (just like when `useMemo` recomputes), and compare the return value in previous _internal slot_, if it’s the same, do nothing, if it changes, update the _internal slot_ and re-render the _parent scope_.

## Details on the design

As this can accept optional dependencies, the question of "what to do if there is an update in the parent component/hook?" should be tackled.

This hook would work like any other hook and follow the rule of hooks. And as it creates a new _call scope_, this doesn’t break the rule of hooks per se (as sub-hooks would not always be called), as the isolated scope would behave like a sub-component.

This hook should only be available in client components, but not in RSCs.

One thing to note: the `callback` doesn’t have to be stable: just like the `reducer` in `React.useReducer`, when there is an update, React should just use its current definition.

## Code examples

### Without dependencies

```jsx
const MyComponent = () => {
  const [index, setIndex] = React.useState(0);
  const [other, setOther] = React.useState(0);
  const fooWithIndex = React.useIsolation(function isolated() {
    const context = React.useContext(CustomHeavyContext);
    return context.foo + index;
  });
};
```

With this piece of code,

- if `index` gets updated, `isolated()` will have to be re-computed
- if `other` gets updated, `isolated()` will have to be re-computed
- if `CustomHeavyContext` gets updated, `isolated()` will have to be re-computed
- if `MyComponent` re-renders for other reason (its parent was updated too for instance), `isolated()` will have to be re-computed

### With dependencies

```jsx
const MyComponent = () => {
  const [index, setIndex] = React.useState(0);
  const [other, setOther] = React.useState(0);
  const fooWithIndex = React.useIsolation(
    function isolated() {
      const context = React.useContext(CustomHeavyContext);
      return context.foo + index;
    },
    [index]
  );
};
```

With this piece of code,

- if `index` gets updated, `isolated()` will have to be re-computed
- if `other` gets updated, `isolated()` **won’t** have to be re-computed
- if `CustomHeavyContext` gets updated, `isolated()` will have to be re-computed

## Advanced example

An example with more realistic code with a _real_ context, props, states

```jsx
const CustomHeavyContext = React.createContext({ foo: [], bar: {}, paz: new Map() });

const MyComponent = (props) => {
  const [otherState] = React.useState({});

  const [arr, setArr] = React.useState([]);
  
  const concatString = React.useIsolation(() => {
    const context = React.useContext(CustomHeavyContext);
    return [...context.foo, ...arr, ...props.otherArr].join(',');
  }, [arr, props.otherArr]);
};
```

In this example: `concatString` will only be recomputed if:
- `CustomHeavyContext` changes,
- `arr` changes,
- `props.otherArr` changes.

But not if `otherState` or other props change.<br>
And if `CustomHeavyContext` changes but `CustomHeavyContext.foo` doesn’t, `concatString` will indeed be recomputed, but the new value will be stable (as `concatString` is a string). So `MyComponent` won’t re-render.

### Similar example with a non-stable value

```jsx
const CustomHeavyContext = React.createContext({ foo: [], bar: {}, paz: new Map() });

const MyComponent = (props) => {
  const [otherState] = React.useState({});

  const [arr, setArr] = React.useState([]);

  const concatArr = React.useIsolation(() => {
    const context = React.useContext(CustomHeavyContext);
    return React.useMemo(() => {
      return [...context.foo, ...arr, ...props.otherArr];
    }, [context.foo, arr, props.otherArr]);
  }, [arr, props.otherArr]);
  // Note: this ^ dep array here is optional as even if `useIsolation` re-runs at each render of `MyComponent`,
  // `arr` & `props.otherArr` are already in the useMemo's dependencies, so React will keep the memoized value within the `useIsolation`
  // and return a stable variable either way
};
```

Here we need to use `useMemo` and as computing concatenation will re-generate a new array every time, even if no array in it doesn’t change.

## Wrapping existing hooks for perf optimizations only

In this section, some pieces of code will be displayed. They won’t be optimal in order to represent what we could find in existing codebases (as not everything can be refactored, often devs have to work with non-optimal code).

Imagine that we have those hooks:

```jsx
const useLongPoll = (url, delay) => {
  const [id, setId] = React.useState(0);

  React.useEffect(() => {
    const intervalId = setInterval(() => setId(id => id + 1), delay);
    return () => clearInterval(intervalId);
  }, []);

  const [status, setStatus] = React.useState();
  React.useEffect(() => {
    const controller = new AbortController();
    fetchStatus(url, { signal: controller.signal }).then(result => setStatus(result));
    return () => controller.abort();
  // Trigger a re-fetch every so often
  }, [id]);

  return status;
}

const MyComponent = () => {
  // Fetch the status every 1s
  const status = useLongPoll('/status', 1000);

  if (!status) {
    return null;
  }
  return <Card status={status}>{getContent(status)}</Card>
}
```

`useLongPoll` isn’t optimal as it creates a re-render every `<delay>`ms. But this may be in one of the dependencies a code base is using so devs may not have the ability of changing that.<br>
This means that `MyComponent` will be re-executed every second (or so) even if the `status` didn’t change. `useIsolation` could fix that:

```jsx
const MyComponent = () => {
  // Fetch the status every 1s
  const status = useIsolation(() => useLongPoll('/status', 1000));

  if (!status) {
    return null;
  }
  return <Card status={status}>{getContent(status)}</Card>
}
```

Now `MyComponent` only re-renders when the status actually changed.

## Settings no dependencies or settings the wrong dependencies

Could using `useIsolation` lead to performance issues if it’s used without dependencies, or with wrong dependencies?

Ideally it shouldn’t, and when following the pseudo-code, there is no reason why it should. Let me explain: the worse-case scenario for performances would be to set no dependencies. In this situation we have 2 possibilities:
1. the returned value isn’t stable (we re-generate a new object for instance at every recomputation)
2. the returned value is stable

For 1., it could be for example this case:

```jsx
const MyComponent = () => {
  const notStable = React.useIsolation(() => {
    return {};
  });
};
```

And in situation, it’d like just like if we were doing this code instead (with a bit of over-head for the memoization / _internal slots_):

```jsx
const MyComponent = () => {
  const notStable = {};
};
```

So it should be okay: it won’t cause performance regressions (at least no dramatic ones), and won’t trigger new re-renders (compared to how `MyComponent` would already behave without `useIsolation`).

For 2., it could be for example this case:

```jsx
const MyComponent = () => {
  const stable = React.useIsolation(() => {
    return React.useContext(MyContext).foo;
  });
};
```

As the returned value is stable (because `React.useContext(MyContext)` will already return the same value / pointer as long as the context didn’t change), it’ll be safe to be used in the render cycle, and even in dependencies of other hooks. Even if `MyComponent` re-renders due to external factors, `stable` won’t change, so it shouldn’t lead to performance issues.

TL;DR: even if no dependencies are set, performances shouldn’t be an issue, and setting them could just further improve performances.

# Drawbacks

The base principle of this new hook is to be able to create new _call scope_ (aka component-like scopes or hooks within hooks).<br>
But this may be a huge change in React’s internals.

As this is deeply related to this "component-like scope", it’s also impossible to polyfill / re-create on the user world and has to be implemented within React (I may be wrong on this).

This hook will also only be available in **client** code, and not in RSC.

One element I didn’t mention is that if `useIsolation` uses variables from the parent scope with the **wrong dependencies**, the hook won’t re-render as expected. As mentioned in [Settings no dependencies or settings the wrong dependencies](#settings-no-dependencies-or-settings-the-wrong-dependencies), as it should be fine to not set dependencies at all, maybe we can remove them. But I feel that they would be a nice addition as you can control re-renders with even finer control.

# Alternatives

As mentioned before, `useContextSelector` proposed in https://github.com/reactjs/rfcs/pull/119 is a good substitute proposal. But this proposal is more generic as it can be also used with any kind of state / variable (see [Wrapping existing hooks for perf optimizations only](#wrapping-existing-hooks-for-perf-optimizations-only).

https://github.com/reactjs/rfcs/issues/168 is a similar RFC for the same hook: `useIsolation`. But this other RFC was closed as it was opened as an issue and not a PR. And this one adds the concept of dependencies to it (otherwise it should be similar).

# Adoption strategy

As this is a new feature, no need to do a breaking change / introduce a new major / do codemods.<br>
It can be released in a minor version.

# How we teach this

The dependency array makes it really close to already existing hooks like `useMemo` / `useCallback` / `useEffect`.

Also this perfectly fits those already existing hooks as it could be built on top of them, so no new patterns to learn. And the previous best-practices can still be applied. It also follows the same rule of hooks as usual.<br>

For new React developers, it could be taught as a hook to boost performance, like `useMemo`: it should work without, but this can prevent unnecessary re-renders.

The only new paradigm is that as hooks will be defined within the `callback`, we’ll need to teach developers that this doesn’t break the rule of hooks (as those would run kind of like in another component).

# Unresolved questions

Is this concurrent-compliant?

Otherwise, I don’t really know.
