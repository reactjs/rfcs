- Start Date: 2020-04-18
- RFC PR:
- React Issue:

# Summary

A new `useKeyedGroups` primitive hook makes it possible to composably consume existing (custom) hooks conditionally or in loops.

# Motivation & Example

A basic example follows, which shows the shortcomings of the existing Rules of Hooks.
Then, a demonstration of how `useKeyedGroups` solves these shortcomings.

## The Setup

Consider the following existing custom hook, which enables any component to listen to use the `EventSource` listener API:

```javascript
function useEventSource(url, callback) {
  const callbackRef = React.useRef(callback);
  React.useEffect(() => {
    callbackRef.current = callback; // (1)
  }, [callback]);
  React.useEffect(() => {
    const es = new EventSource(url);
    es.addEventListener("message", (e) => {
      callbackRef.current(e);
    });
    return () => {
      es.close(); // (2)
    };
  }, [url]); // (3)
}
```

There are three important correctness properties that this custom hook provides:

1. When the callback is updated across renders, the new callback will always be called
2. When the URL changes (or the component unmounts), the EventSource resource will be closed
3. When the URL _doesn't_ change, the existing EventSource will be reused, and not redundantly closed and re-opened

It provides a simple, declarative API which can be used in the following manner:

```javascript
const InteractionList = ({ url }) => {
  const [interactions, addInteraction] = React.useReducer(
    (state, addInteraction) => [...state, addInteraction],
    []
  );

  useEventSource(`${url}/comments`, (e) => {
    addInteraction({ name: `${url} comment`, data: e.data });
  });
  useEventSource(`${url}/reactions`, (e) => {
    addInteraction({ name: `${url} react`, data: e.data });
  });

  return (
    <ul>
      {interactions.map(({ name, data }, index) => (
        <li key={index}>
          {name}: {data}
        </li>
      ))}
    </ul>
  );
};
```

Custom hooks make reusing logic very easy, composable, and safe.
However, they have to obey the laws of hooks: `useEventSource` must be called a fixed number of times within our `<InteractionList/>`.

## The Problem

UI requirements change, and the `<InteractionList urls={["/abc", "/def", "/ghi"]} />` now needs to consume and combine events from multiple streams!

Since we now need to consume from a _dynamic collection of URLs_, we can no longer use our `useEventSource` hook!
This means the nice properties (1), (2), and (3) that we previously got "for free" by writing an idiomatic hook are no longer available to use.

We either have to split our one component into multiple, which may involve a restructuring of our application, or we have to come up with a brand-new (much, much more complicated) `useMultipleEventSource` custom hook.

**We lose the composability of custom hooks when they cannot be used conditionally or in loops.**
This instead forces us to restructure applications around the kinds of custom hooks we're able to write, instead of structuring it around the "natural" way to use them.

**But the Rules of Hooks exist for good reason.**
Hence, we need a special escape-hatch (`useKeyedGroups`) to be able to re-use our custom hook.

## The Proposed Solution

This proposal suggests the creation of a new primitive hook: `useKeyedGroups`.
This hook solves all the problems thus-far introduced:

- We can continue to use `useEventSource` without needing to change its implementation
- `InteractionList` keeps its same general shape, only changing the parts that matter (looping over all the URLs we want)
- All custom hooks remain composable, independent, and easy-to-understand (we only break the rules within the `useKeyedGroups` call itself; and this isn't observable from "above")

```javascript
const InteractionList = ({ urls }) => {
  const [interactions, addInteraction] = React.useReducer(
    (state, addInteraction) => [...state, addInteraction],
    []
  );

  React.useKeyedGroups((withKey) => {
    for (const url of urls) {
      withKey(url, () => {
        useEventSource(`${url}/comments`, (e) => {
          addInteraction({ name: `${url} comment`, data: e.data });
        });
        useEventSource(`${url}/reactions`, (e) => {
          addInteraction({ name: `${url} react`, data: e.data });
        });
      });
    }
  });

  return (
    <ul>
      {interactions.map(({ name, data }, index) => (
        <li key={index}>
          {name}: {data}
        </li>
      ))}
    </ul>
  );
};
```

We are able to continue to use our `<InteractionList />` implementation with minimal changes, along with the existing `useEventSource` custom hook.
The solution is also composable, so we can also easily create a `useMultipleEventSource` hook without duplicating code for making actual connections:

```javascript
function useMultipleEventSource(urls, callback) {
  React.useKeyedGroups((withKey) => {
    for (const url of urls) {
      withKey(url, () => {
        useEventSource(url, (e) => callback(url, e));
      });
    }
  });
}
```

In this way, **`useKeyedGroups` makes custom hooks even more composable by default**.
No existing application or library code needs to change to make this happen, and no assumptions that existing applications or libraries make will be broken (provided that the new hook is used correctly).

The details for `useKeyedGroup`'s semantics follow.

# Detailed design

The new `useKeyedGroups` primitive hook takes a 1-argument function parameter.
We will call this the _Groups Block_:

```javascript
React.useKeyedGroups((withKey) => {
  // The "Groups Block"
  // No hooks in here; the regular Rules apply
});
```

The _Groups Block_ is called immediately (synchronously) and its return value is returned from `useKeyedGroups`.
It's provided with the special `withKey` callback _that can only be called from within the Groups Block_.
`withKey` is passed as a parameter to make it deliberately difficult to accidentally call it outside of this block, since this is always erroneous.

`withKey` is called with a `key` (which is a string or number, like component keys) and a "Keyed Hook Block":

```javascript
React.useKeyedGroups((withKey) => {
  const [countYes, addYes] = withKey("yes", () => {
    // "yes" Keyed Hook Block
    const [countYes, setCountYes] = React.useState(0);
    return [countYes, () => setCountYes(countYes + 1)];
  });

  // ...
});
```

Each _Keyed Hook Block_ may call hooks, just like a custom hook or any component.
The _Keyed Hook Block_ is invoked immediately (synchronously) and its return value is returned from `withKey`.

Before the _Keyed Hook Block_ executes, `withKey` manipulates the global memory attached to the current fiber so that hooks will find the memory associated with the current key.
To make this possible, `useKeyedGroups` stores a map from each key to the memory cells needed for all the hooks called when that key is used.
After the Keyed Hook Block returns, `withKey` restores the fiber memory to prevent hooks from being called again until the _Groups Block_ finishes or another _Keyed Hook Block_ begins.

When locating/assigning memory for a key, there are three possible cases:

- The key is _novel_: it did not appear in the previous render (or there was no previous render). In this case, all hooks behave as in "mount" mode (e.g. `useState` and `useRef` will reinitialize their state).
- The key is _repeated_: it did appear in the previous render; in this case, each hook will receive the same memory as in the previous render (even if the `withKey` was called in a different order than the previous render)
- The key is _missing_: this can only be detected at the end of the _Groups Block_. Any keys that were present on the previous render but are absent now will need to be cleaned up; their states will be lost and their cleanup functions will run (just as if the component was unmounted).

Since `useKeyedGroups` and `withKey` are responsible for updating/managing hook memory, only very minimal changes should be requires for the other primitive hooks.

## Behavior with Reordered Keys

`withKey` callbacks can be invoked in different orders between renders. For example, it's perectly legal to have

```javascript
React.useKeyedGroups((withKey) => {
  withKey("FIRST", () => {
    // ...
  });
  withKey("SECOND", () => {
    // ...
  });
});
```

in one render, and then

```javascript
React.useKeyedGroups((withKey) => {
  withKey("SECOND", () => {
    // ...
  });
  withKey("FIRST", () => {
    // ...
  });
});
```

in a second.
When this happens, the _key identities_, and not the order of `withKey` calls, determines which state the hooks inherit from the previous render.

`useEffect`/`useLayoutEffect` run callbacks asynchronously, and so might depend on their order that they are invoked.
For consistency, their effects should run in the order in which they are invoked in the current render.
Then their cleanup functions will run in the order that they were initially rendered, much like today.
(It's unlikely most applications will care about the order of these events)

## Behavior with Duplicate Keys

It is an error to pass the same key twice to `withKey` in the same render in the same _Groups Block_.

# Drawbacks

## Implementation & Restrictions on Future Design

The implementation may impose new requirements on the architecture for the scheduler, since `useKeyedGroups` enables more expressive orderings for the other hooks.
Hence, this _may_ include changes to the way other hooks are implemented, in particular how they access their persistent state. A prototype implementation would likely determine whether this effects are likely to be significant.

## Implementation in Userspace

`useKeyedGroups` cannot be implemented in userspace, since any implementation violates the laws of hooks.

## Impact on Teaching

The `useKeyedGroups` hook would be an advanced feature, intended for users wanting to get the most expressivity out of custom hooks as possible.
It makes the complete "rules of hooks" slightly more complicated, though they aren't affected in cases where `useKeyedGroups` is invisible (for example, a custom hook that happens to use `useKeyedGroups` in its implementation doesn't behave noticeably differently when it is used than any other custom hook).

There is a superficial similarity to `useEffect` which may be confusing (both take callback parameters, but `useKeyedGroups` invokes its synchronously instead of asynchronously).

`useKeyedGroups`'s interface is deliberately somewhat clunky - it should be reached for as a last resort, to make custom hooks work well together or with dynamic data.

## Tooling

Linters for the Rules of Hooks would need to be updated to account for the new behavior.
The new behavior is slightly more complex, but can still be linted statically without much additional complexity.

## Cost of Migrating Applications

Existing applications will continue to work unaffected.
Some applications may be able to eliminate redundant code in custom hooks.

# Alternatives

## Do Nothing

Most applications do not benefit from `useKeyedGroups`.
Some applications could benefit, but can also be restructured to avoid needing it.

## Add specific `useHookLoop` or `useHookIf` primitive hooks

The design of `useKeyedGroups` makes it possible to perform arbitrary control-flow to decide when, how, and if, any custom/primitive hooks should be invoked.
This means that it is one primitive which covers many cases.

It also means that it provides much more power than most users will need; `useHookLoop(list, useHook)` and `useHookIf(condition, useHook)` would cover _most_ (but probably not _all_) cases.
Teaching them would also likely be simpler.

However, these are easy to provide as a library (either directly from React or through a third-party package), and cannot accomplish certain control structures:

- Run hooks depending on the return values of previous hooks in the same (dynamic) group
- Pass results from one hook into others within the same (dynamic) group

# Adoption strategy

Existing application or library code would not need to be changed.

# How we teach this

The best way to teach `useKeyedGroups` would be as a way to break out of the "rules of hooks" when they are too stringent.

In particular, it should probably not be presented until after the rules themselves, since they still apply within each `withKey` argument function.

Some libraries may update to make use of it, or be restructured to encourage its use.

# Unresolved questions

- What is the best name for this hook?
- What is the best name for the `withKey` callback? Nothing will care about the name of this callback, but setting forth a consistent, easy-to-understand convention is a good idea
