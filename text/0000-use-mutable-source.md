- Start Date: 2020-02-13
- RFC PR: [18000](https://github.com/facebook/react/pull/18000)
- React Issue: N/A

# useMutableSource

`useMutableSource()` enables React components to **safely** and **efficiently** read from a mutable external source in Concurrent Mode. The API will detect mutations that occur during a render to avoid tearing and it will automatically schedule updates when the source is mutated.

# Basic example

This hook is designed to support a variety of mutable sources. Below are a few example cases.

### Redux stores

`useMutableSource()` can be used with Redux stores:

```js
// May be created in module scope, like context:
const reduxSource = createMutableSource(
  store,
  // Because the state is immutable, it can be used as the "version".
  () => reduxStore.getState()
);

// Redux state is already immutable, so it can be returned as-is.
// Like a Redux selector, this method could also return a filtered/derived value.
//
// Because this method doesn't require access to props,
// it can be declared in module scope to be shared between components.
const getSnapshot = store => store.getState();

// Redux subscribe method already returns an unsubscribe handler.
//
// Because this method doesn't require access to props,
// it can be declared in module scope to be shared between components.
const subscribe = (store, callback) => store.subscribe(callback);

function Example() {
  const state = useMutableSource(reduxSource, getSnapshot, subscribe);

  // ...
}
```

### Browser APIs

`useMutableSource()` can also read from non traditional sources, e.g. the shared Location object, so long as they can be subscribed to and have a "version".

```js
// May be created in module scope, like context:
const locationSource = createMutableSource(window, {
  // Although not the typical "version", the href attribute is stable,
  // and will change whenever part of the Location changes,
  // so it's safe to use as a version.
  getVersion: () => window.location.href
});

// Because this method doesn't require access to props,
// it can be declared in module scope to be shared between components.
const getSnapshot = window => win.location.pathname;

// This method can subscribe to root level change events,
// or more snapshot-specific events.
// In this case, since Example is only reading the "friends" value,
// we only have to subscribe to a change in that value
// (e.g. a "friends" event)
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
const userDataSource = createMutableSource(store, {
  getVersion: () => data.version
});

// This method can subscribe to root level change events,
// or more snapshot-specific events.
// In this case, since Example is only reading the "friends" value,
// we only have to subscribe to a change in that value
// (e.g. a "friends" event)
//
// Because this method doesn't require access to props,
// it can be declared in module scope to be shared between components.
const subscribe = (data, callback) => {
  data.addEventListener("friends", callback);
  return () => data.removeEventListener("friends", callback);
};

function Example({ onlyShowFamily }) {
  // Because the snapshot depends on props, it has to be created inline.
  // useCallback() memoizes the function though,
  // which lets useMutableSource() know when it's safe to reuse a snapshot value.
  const getSnapshot = useCallback(
    data =>
      data.friends
        .filter(
          friend => !onlyShowFamily || friend.relationshipType === "family"
        )
        .friends.map(friend => friend.id),
    [onlyShowFamily]
  );

  const friendIDs = useMutableSource(userDataSource, getSnapshot, subscribe);

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
  subscribe: (source: Source, callback: Function) => () => void
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

Tracking a source's version allows us to avoid tearing during a mount (before our component has subscribed to the source). Whenever a mounting component reads from a mutable source, this number should be checked to ensure that either (1) this is the first mounting component to read from the source during the current render or (2) the version number has not changed since the last read. A changed version number indicates a change in the underlying store data, which may result in a tear.

Like Context, this hook should support multiple concurrent renderers (e.g. ReactDOM and ReactART, React Native and React Fabric). To support this, we will track two work-in-progress versions (one for a "primary" renderer and one for a "secondary" renderer).

This value should be reset either when a renderer starts a new batch of work or when it finishes (or discards) a batch of work. This information could be stored:

- On each mutable source itself in a primary and secondary field.
  - **Con**: Requires a separate array/list to track mutable sources with outstanding changes.
- At the module level as a `Map` of mutable source to pending primary and secondary version numbers.
  - **Con**: Requires at least one extra `Map` structure.

> ⚠️ **Decision** Store versions directly on the source itself and track pending changes with an array.

#### Pending update expiration times

Tracking pending update times enables already mounted components to safely reuse cached snapshot values without tearing in order to support higher priority updates. During an update, if the current render’s expiration time is **≤** the stored expiration time for a source, it is safe to read new values from the source. Otherwise a cached snapshot value should be used temporarily<sup>1</sup>.

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

- The user-provided config object (with getter functions).
- The latest (cached) snapshot value.
- The mutable source itself (in order to detect if a new source is provided).
- A destroy function (to unsubscribe from a source)

### Scenarios to handle

#### Initial mount (before subscription)

When a component reads from a mutable source that it has not yet subscribed to<sup>1</sup>, React first checks to see if there are any pending updates for the source already scheduled on the current root.

- ✗ If there is a pending update and the current expiration time is **>** the pending time, the read is **not safe**.
  - Throw and restart the render.
- If there are no pending updates, or if the current expiration time is **≤** the pending time, has the component already subscribed to this source?
  - ✓ If yes, the read is **safe**.
    - Store the snapshot value on `memoizedState`.
  - If no, the the read **may be safe**.

For components that have not yet subscribed to their source, React reads the version of the source and compares it to the tracked work-in-progress version numbers.

- ✓ If there is no recorded version, this is the first time the source has been used. The read is **safe**.
  - Record the current version number (on the root) for later reads during mount.
  - Store the snapshot value on `memoizedState`.
- ✓ If the recorded version matches the store version used previously, the read is **safe**.
  - Store the snapshot value on `memoizedState`.
- ✗ If the recorded version is different, the read is **not safe**.
  - Throw and restart the render.

¹ This case could occur during a mount or an update (if a new mutable source was read from for the first time).

#### Mutation

React will subscribe to sources after commit so that it can schedule updates in response to mutations. When a mutation occurs<sup>1</sup>, React will calculate an expiration time for processing the change, and will:

- Schedule an update for that expiration time.
- Update a root level entry for this source to specify the next scheduled expiration time.
  - This enables us to avoid tearing within the root during subsequent renders.

¹ Component subscriptions may only subscribe to parts of the external source they care about. Updates will only be scheduled for component’s whose subscriptions fire.

#### Update (after subscription)

React will eventually re-render when a source is mutated, but it may also re-render for other reasons. Even in the event of a mutation, React may need to render a higher priority update before processing the mutation. In that case, it’s important that components do not read from a changed source since it may cause tearing.

In order to process updates safely, React will track pending root level expiration times per source.

- ✓ If the current render’s expiration time is **≤** the stored expiration time for a source, it is **safe** to read.
  - Store an updated snapshot value on `memoizedState`.
- If the current render expiration time is **>** than the root priority for a source, consider the config object.
  - ✓ If the config object has not changed, we can re-use the **cached snapshot value**.<sup>1</sup>
  - ✗ If the config object has changed, the **cached snapshot is stale**.
    - Throw and restart the render.

¹ React will later re-render with new data, but it’s okay to use a cached value if the memoized config has not changed- because if the inputs haven’t changed, the output will not have changed.

#### React render new subtree

React may render a new subtree that reads from a source that was also used to render an existing part of the tree. The rules for this scenario is the same as the initial mount case described above.

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
