- Start Date: 2020-02-13
- RFC PR: [18000](https://github.com/facebook/react/pull/18000)
- React Issue: N/A

# useMutableSource

`useMutableSource()` enables React components to **safely** and **efficiently** read from a mutable external source in Concurrent Mode. The API will detect mutations that occur during a render to avoid tearing and it will automatically schedule updates when the source is mutated.

# Basic example

This hook is designed to support a variety of mutable sources. Below are a few example cases.

### Browser APIs

`useMutableSource()` can also read from non traditional sources, e.g. the shared Location object, so long as they can be subscribed to and have a "version".

```js
// May be created in module scope, like context:
const locationSource = createMutableSource(
  window,
  // Although not the typical "version", the href attribute is stable,
  // and will change whenever part of the Location changes,
  // so it's safe to use as a version.
  () => window.location.href
);

// Because this method doesn't require access to props,
// it can be declared in module scope to be shared between components.
const getSnapshot = window => window.location.pathname;

// This method can subscribe to root level change events,
// or more snapshot-specific events.
//
// Because this method doesn't require access to props,
// it can be declared in module scope to be shared between components.
const subscribe = (window, callback) => {
  window.addEventListener("popstate", callback);
  return () => window.removeEventListener("popstate", callback);
};

function Example() {
  const pathName = useMutableSource(locationSource, getSnapshot, subscribe);

  // ...
}
```

### Selectors that use props

Sometimes a state value is derived using component `props`. In this case, `useCallback` should be used to keep the snapshot and subscribe functions stable.

```js
// May be created in module scope, like context:
const userDataSource = createMutableSource(userData, () => userData.version);

// This method can subscribe to root level change events,
// or more snapshot-specific events.
// In this case, since Example is only reading the "friends" value,
// we only have to subscribe to a change in that value
// (e.g. a "friends" event)
//
// Because this method doesn't require access to props,
// it can be declared in module scope to be shared between components.
const subscribe = (userData, callback) => {
  userData.addEventListener("friends", callback);
  return () => userData.removeEventListener("friends", callback);
};

function Example({ onlyShowFamily }) {
  // Because the snapshot depends on props, it has to be created inline.
  // useCallback() memoizes the function though,
  // which lets useMutableSource() know when it's safe to reuse a snapshot value.
  const getSnapshot = useCallback(
    userData =>
      userData.friends
        .filter(
          friend => !onlyShowFamily || friend.relationshipType === "family"
        )
        .map(friend => friend.id),
    [onlyShowFamily]
  );

  const friendIDs = useMutableSource(userDataSource, getSnapshot, subscribe);

  // ...
}
```

### Redux stores

Redux users would likely never use the `useMutableSource` hook directly. They would use a hook provided by Redux that uses `useMutableSource` internally.

##### Mock Redux implementation
```js
// Somewhere, the Redux store needs to be wrapped in a mutable source object...
const mutableSource = createMutableSource(
  reduxStore,
  // Because the state is immutable, it can be used as the "version".
  () => reduxStore.getState()
);

// It would probably be shared via the Context API...
const MutableSourceContext = createContext(mutableSource);

// Because this method doesn't require access to props,
// it can be declared in module scope to be shared between hooks.
const subscribe = (store, callback) => store.subscribe(callback);

// Oversimplified example of how Redux could use the mutable source hook:
function useSelector(selector) {
  const mutableSource = useContext(MutableSourceContext);

  const getSnapshot = useCallback(store => selector(store.getState()), [
    selector
  ]);

  return useMutableSource(mutableSource, getSnapshot, subscribe);
}
```

#### Example user component code

```js
import { useSelector } from "react-redux";

function Example() {
  // The user-provided selector should be memoized with useCallback.
  // This will prevent unnecessary re-subscriptions each update.
  // This selector can also use e.g. props values if needed.
  const memoizedSelector = useCallback(state => state.users, []);
  
  // The Redux hook will connect user code to useMutableSource.
  const users = useSelector(memoizedSelector);

  // ...
}
```

### Observables

Observables don’t have an intrinsic version number and so they are incompatible with this API. It might be possible to add a derived version number to an observable, as shown below, but **it would not be safe to do this during render** without causing potential memory leaks.

```js
function createBehaviorSubjectWithVersion(behaviorSubject) {
  let version = 0;

  const subscription = behaviorSubject.subscribe(() => {
    version++;
  });

  return new Proxy(behaviorSubject, {
    get: function(object, prop, receiver) {
      if (prop === "version") {
        return version;
      } else if (prop === "destroy") {
        return () => subscription.unsubscribe();
      } else {
        return object[prop];
      }
    }
  });
}
```

# Motivation

The current best "alternates" to this API are the [Context API](https://reactjs.org/docs/context.html) and [`useSubscription` hook](https://www.npmjs.com/package/use-subscription).

### Context API

The Context API is not currently suited for sources that are used by many components throughout the tree, as changes to the context result in updates that are very heavy (for example, see [Redux v6 performance challenges](https://github.com/reduxjs/react-redux/issues/1177)). (There are currently proposals to improve this: [RFC 118](https://github.com/reactjs/rfcs/pull/118) and [RFC 119](https://github.com/reactjs/rfcs/pull/119).)

### `useSubscription`

[This Gist](https://gist.github.com/bvaughn/054b82781bec875345bd85a5b1344698) outlines the differences between `useMutableSource` and `useSubscription`. The main advantages of this new API are:

- No temporary tearing will occur during render (even before the initial subscription).
- Subscriptions can be "scoped" so that updates to parts of a mutable source only impact the relevant components (and not all components reading from the source). This means that in the common case, this hook should perform much better.

# Detailed design

`useMutableSource` is similar to [`useSubscription`](https://github.com/facebook/react/tree/master/packages/use-subscription).

- Both require a memoized “config” object with callbacks to read values from an external “source”.
- Both require a way to subscribe and unsubscribe to the source.

There are some differences though:

- `useMutableSource` requires the source as an explicit parameter. (React uses this value to protect against "tearing" and ensure that all components reading from a particular source render with the same version of data.)
- `useMutableSource` requires values read from the source to be immutable snapshots. This enables values to be reused during high priority render, allowing more expensive re-renders to be deferred when needed.

### Public API

```js
type MutableSource<Source> = {|
  /*…*/
|};

function createMutableSource<Source>(
  source: Source,
  getVersion: () => $NonMaybeType<mixed>
): MutableSource<Source> {
  // ...
}

function useMutableSource<Source, Snapshot>(
  source: MutableSource<Source>,
  getSnapshot: (source: Source) => Snapshot,
  subscribe: (source: Source, callback: () => void) => () => void
): Snapshot {
  // ...
}
```

## Implementation

### Root or module scope changes

Mutable source requires tracking two pieces of info at the module level:

1. Work-in-progress version number (tracked per source, per renderer)
1. Pending update expiration times (tracked per root, per source)

#### Version number

Tracking a source's version allows us to avoid tearing when reading from a source that a component has not yet subscribed to.

In this case, the version should be checked to ensure that either:
1. This is the first mounting component to read from the source during the current render, or
2. The version number has not changed since the last read. (A changed version number indicates a change in the underlying store data, which may result in a tear.)

Like Context, this hook should support multiple concurrent renderers (e.g. ReactDOM and ReactART, React Native and React Fabric). To support this, we will track two work-in-progress versions (one for a "primary" renderer and one for a "secondary" renderer).

This value should be reset either when a renderer starts a new batch of work or when it finishes (or discards) a batch of work. This information could be stored:

- On each mutable source itself in a primary and secondary field.
  - **Con**: Requires a separate array/list to track mutable sources with outstanding changes.
- At the module level as a `Map` of mutable source to pending primary and secondary version numbers.
  - **Con**: Requires at least one extra `Map` structure.

> ⚠️ **Decision** Store versions directly on the source itself and track pending changes with an array.

#### Pending update expiration times

Tracking pending updates per source enables newly-mounting components to read without potentially conflicting with components that read from the same source during a previous render.

During an update, if the current render’s expiration time is **≤** the stored expiration time for a source, it is safe to read new values from the source. Otherwise a cached snapshot value should be used temporarily<sup>1</sup>.

When a root is committed, all pending expiration times that are **≤** the committed time can be discarded for that root.

This information could be stored:

- On each Fiber root as a `Map` of mutable source to pending update expiration time.
- On each mutable source as a `Map` of Fiber root to pending update expiration time.
  - **Con**: Requires a separate data structure to map roots to mutable sources with outstanding changes (since outstanding changes are cleaned up per-root on commit).

> ⚠️ **Decision** Store pending update times in a `Map` on the Fiber root.

<sup>1</sup> Cached snapshot values can't be reused when a config changes between render. More on this below...

#### A word about why both pending expiration and version are required

Although useful for updates, pending update expiration times are not sufficient to avoid tearing for newly mounted components even if the source has already been used by another component. Since each component may subscribe to a different part of the store, the following scenario is possible:

1. Some components mount and subscribe to source A.
2. React starts a new render.
3. A new component (not previously mounted) reads from source A, and then React yields.
4. Source A changes in a way that does not notify any of the currently-subscribed components, but would impact the new component (which is not yet subscribed).
5. Another new component (not previously mounted) reads from source A. At this point, there are no pending updates for the source, but it has changed and so reading from it may cause a tear.

### Hook state

The `useMutableSource()` hook’s memoizedState will need to track the following values:

- The user-provided `getSnapshot` and `subscribe` functions.
- The latest (cached) snapshot value.
- The mutable source itself (in order to detect if a new source is provided).
- The (user-returned) unsubscribe function

### Scenarios to handle

#### Reading from a source before subscribing

When a component reads from a mutable source that it has not yet subscribed to<sup>1</sup>, React first checks the version number to see if anything else has read from this source during the current render.

- If there is a recorded version number (i.e. this is not the first read) does it match the source's current version?
  - ✓ If both versions match, the read is **safe**.
    - Store the snapshot value on `memoizedState`.
  - ✗ If the version has changed, the read is **not safe**.
    - Throw and restart the render.

If there is no version number, the the read **may be safe**. We'll need to next check pending updates for the source to determine this.

- ✓ If there are no pending updates the read is **safe**.
  - Store the snapshot value on `memoizedState`.
  - Store the version number for subsequent reads during this render.
- ✓ If the current expiration time is **≤** the pending time, the read is **safe**.
  - Store the snapshot value on `memoizedState`.
  - Store the version number for subsequent reads during this render.
- ✗ If the current expiration time is **>** the pending time, the read is **not safe**.
  - Throw and restart the render.

<sup>1</sup> This case could occur during a mount or an update (if a new mutable source was read from for the first time).

#### Reading from a source after subscription

React will eventually re-render when a source is mutated, but it may also re-render for other reasons. Even in the event of a mutation, React may need to render a higher priority update before processing the mutation. In that case, it’s important that components do not read from a changed source since it may cause tearing.

In the event the a component renders again without its subscription firing (or as part of a high priority update that does not include the subscription change) it will typically be able to re-use the cached snapshot.

The one case where this will not be possible is when the `getSnapshot` function has changed. Snapshot selectors that are dependent on `props` (or other component `state`) may change even if the underlying source has not changed. In that case, the cached snapshot is not safe to reuse, and `useMutableSource` will have to throw and restart the render.

# Design constraints

- Tearing guarantees are only enforced within a root, for components using the same MutableSource value. Tearing between roots is possible.
- Values read and returned from the store must be immutable in the same way as e.g. class state or props objects.
  - e.g. ✓ `getSnapshot: source => Array.from(source.friendIDs)`
  - e.g. ✗ `getSnapshot: source => source.friendIDs`
  - Values don't need to literally be immutable but should at least be cloned so they are disconnected from the store and are not mutated by changes to the external source.

* Mutable source must have some form of stable version.
  - Version should be global (for the entire source, not parts of the source).
    - e.g. ✓ `getVersion: () => source.version`
    - e.g. ✗ `getVersion: () => source.user.version`
  - Version should change whenever any part of the source is mutated.
  - Version does not have to be a number or even a single attribute.
    - It can be a serialized form of the data, so long as it is stable and unique. (For example, reading query parameters might treat the entire URL string as the version.)
    - It can be the state itself, if the value is immutable (e.g. a Redux store is mutable, but its state is immutable).

# Alternatives

See "Motivation" section above.

# Adoption strategy

This hook is primarily intended for use by libraries like Redux (and possibly Relay). Work with the maintainers of those libraries to integrate with the hook.

# How we teach this

New [reactjs.org](https://reactjs.org/) documentation and blog post.

# Unresolved questions

- Are there any common/important types of mutable sources that this proposal will not be able to support?
