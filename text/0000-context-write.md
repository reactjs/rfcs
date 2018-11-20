- Start Date: 2018-11-19
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Proposes an extension to the context API for updating the default context value. This would allow for React-managed state that lives outside the UI tree and is shared across roots.

# Basic example

```js
const Context = React.createContext(initialValue, contextDidUpdate);
// Update the global context value across all roots. Any context consumer that
// is not wrapped in a Provider will re-render with this value.
Context.write(newValue);
// Functional updates also supported for access to the previous value.
Context.write(prevValue => newValue);

function contextDidUpdate(newContext) {
  // Optional callback that fires whenever context changes
}
```

# Motivation

### Caching external data

Where should a React app store data fetched over IO?

It may be tempting to cache it in component state. This is a common pattern for pre-Suspense React apps. But there are some drawbacks. For example, if a Comment component depends on data from the server, consider the implications of storing that data in the Comment component's local state. What if the same comment data is rendered by a different component on another part the page? I shouldn't have to fetch the same comment twice. What if I navigate away and my comment is unmounted, but then I later navigate back? I don't want to refetch the same comment again merely because the old instance was unmounted. The underlying principle here is that data fetched over IO typically does not semantically align with the lifetime of a component used to present that data. The data and its presentation are separate concepts.

Still, although caching in component state isn't ideal, it's arguably "good enough" for many use cases today.

Not so with Suspense. The Suspense model is that the first time React renders an IO-bound component, an exception is thrown when attempting to read the data. While React is waiting for the data to load, it continues rendering the rest of the tree, but it doesn't commit the result; the partially completed tree is _discarded_. Once the data has resolved, React attempts to render the entire tree over again from scratch. It may, as an optimiztion, reuse parts of the previous attempt. But it's not a guarantee. That means using local component state to cache data won't work, because that state will most likely not be persisted. The underlying priciple here is that you can't cache the intermediate results of a computation (server data) on the output of the computation itself (the React tree).

One might argue our Suspense implementation is wrong and we should change it. However, the throw-it-out-and-try-again design is not primarily an implementation decision, but a modeling one. The main motivating use case is server rendering. If data were cached using the component tree, then the server would need to track the state of every component. By moving to an external cache, the React server renderer can remain stateless; all it needs to track is which parts of the tree have yet to complete. Caching on the component tree would also complicate client-side hydration. The server renderer would need to serialize the tree of component state and send it to the client. Not only is this complicated, it's bad practice. It's better to hydrate that data into in a normalized data cache, anyway, for the reasons described above.

Ok, so storing in component state won't work with Suspense. Where then? React currently doesn't have a good answer to this question.

Caching in a global singleton, or some other object that's not managed by React, will work up to a point. But it's tricky to do correctly in concurrent mode without inconsistencies (tearing). Even if you manage to do that, you can't take full advantage of concurrent rendering, where there may be interleaved pending updates, and persistent mutations are only permitted in the commit phase.

React does provide an API that's similar to what we want: context. Context is already used to read values that semantically live outside the React tree. But many uses of context today are in fact backed by global singletons. We're missing a way to manage external state in an idiomatic way that works with concurrent rendering and Suspense.

## Automatic dependency injection

One of the most common uses of context in React is for dependency injection. Instead of relying on global mutable state, a framework may broadcast a value to a subtree using the context API. This preserves the option to wrap a tree of components in a nested context provider without changing the consumers. In practice, this means many React apps have a section near the root that contains all the provider components needed to render the app. Not only is this tedious, but it also means React has to load all the code for those libraries up front, even if the consumers haven't mounted yet.

The new context API partially addresses this problem by allowing for a default context value. Any consumer that reads from context but is not wrapped in a provider receives the default value. But there's no mechanism to update that value, so it's insufficient for any type of context that is stateful. The only way to do this today is to wrap the app in a stateful component and pass state to a context provider.

## Sharing state across roots

Wrapping a tree with a stateful context provider component doesn't address apps that are comprised of multiple roots. Each root needs its own provider. If the data is stored locally in the provider component, then each root will have a separate cache, leading to duplicate requests. If you move the state outside of the providers to an external store, then the providers will need to subscribe to the store's updates.

An external store's state is also not managed by React, so it can't be fully compatible with concurrent rendering. Short of full compatibility, even limited compatibility without tearing is difficult to implement correctly without deep knowledge of React's rendering model. An idiomatic API designed for this use case should at the very least make it possible to lift state out of React without causing tearing and without always falling back to synchronous mode.

# Detailed design

## Motivating examples

### Immutable store (like React Redux)

Implementing immutable stores is straightfoward. Actions are applied in the render phase using the functional form of `Context.write`.

```js
const Store = React.createContext(initialState);

export function dispatch(action) {
  Store.write(state => reducer(state, action));
}

export function useStore() {
  return useContext(Store);
}
```

### Mutable store (like React Cache)

Mutable stores are a bit trickier but can still work. The trick is to avoid mutating the store directly. Instead, add pending mutations to a queue, and only flush the queue once you reach the commit phase (using the `contextDidUpdate` callback).

In this example, the context value for the cache is comprised of two maps: a cached map, and pending map. Components read from the pending map first, before falling back to the cached map. The pending map is immutable: it's populated by `Context.write`. The cached map is mutable: it's mutated in the commit phase.

The advantage of this approach is that only the pending map needs to be immutable (copy-on-write). Because the pending map is a small subset of the entire cache, this minimizes the amount of copying needed to support concurrent access.

```js
const Cache = React.createContext(
  {cachedRecords: null, pendingRecords: null},
  cacheDidCommit,
);

export function read(key) {
  const cache = Cache.read();
  const {cachedRecords, pendingRecords} = cache;

  let record;
  if (pendingRecords !== null && pendingRecords.has(key)) {
    // Always read from pending map first
    record = pendingRecords.get(key);
  } else if (cachedRecords !== null && cachedRecords.has(key)) {
    // If there's no pending update, read from cached map.
    record = cachedRecords.get(key);
  } else {
    // If there's no match, create a new, empty record.
    record = {tag: 'pending', value: null};
  }

  switch (record.tag) {
    case 'pending':
      // This initiates a request and throws a promise to suspend the render.
      suspendOnPendingRecord(cache, record);
    case 'resolved':
      // Return the cached value.
      return record.value;
    case 'rejected':
      // Throw an error.
      throw record.value;
  }
}

export function invalidateByKey(key) {
  // Create an empty record.
  const newRecord = {tag: 'pending', value: null};

  // Schedule an update to overwrite the cached record with the new one.
  Cache.write(cache => {
    const {cachedRecords, pendingRecords} = cache;
    if (cachedRecords === null || !cachedRecords.has(key)) {
      // If there's not already a cached value, then there's nothing to
      // invalidate. Reuse the existing cache.
      return cache;
    }
    // Add to the pending records map. This needs to be an immutable operation:
    // we copy the previous map before setting. The cache itself isn't updated
    // until the commit phase.
    pendingRecords = new Map(pendingRecords);
    pendingRecords.set(key, newRecord);
    return {cachedRecords, pendingRecords};
  });
}

export function invalidateAll() {
  // Clear the entire cache.
  Cache.write({cachedRecords: null, pendingRecords: null});
}

function cacheDidCommit(committedCache) {
  // Now that we've reached the commit phase, it's safe to mutate. Collapse the
  // two maps into one by overwriting the cached map with the values from the
  // pending map.
  const pendingRecords = committedCache.pendingRecords;
  if (pendingRecords !== null) {
    const cachedRecords = committedCache.cachedRecords;
    if (cachedRecords === null) {
      cache.cachedRecords = pendingRecords;
    } else {
      pendingRecords.forEach((record, key) => {
        cachedRecords.set(key, record);
      });
    }
    // The pending records have been persisted, so we no longer need them.
    cache.pendingRecords = null;
  }
}
```

# Drawbacks

## How to handle multiple roots

The trickiest question is how to deal with multiple roots. React does not guarantee consistency across roots; each root has its own commit phase, and suspending inside one root has no effect on the others. This isn't observable in synchronous mode, since React will block the main thread (including paint) until every root's commit phase has finished. In concurrent mode, however, React may yield in between each commit. With Suspense, the time between each commit phase could vary by many seconds.

This model has several consequences. For each context, React would have to maintain a separate version per root, as well as a separate queue of updates. The `contextDidUpdate` callback complicates this further. If each root updates separately, it only makes sense to fire the callback once every root has committed, which necessitates reference counting or some similar tracking mechanism.

An alternative model is to treat all the roots as siblings and commit them at the same time. This could be viable for single page React apps (where there aren't that many roots, anyway). It doesn't work so well in cases where React is embedded inside another framework (e.g. progressive enhancement of a server rendered app), where temporary inconsistencies may be desirable. For example, the opt-in API for concurrent mode relies on roots committing separately so that you can upgrade some roots to concurrent mode without upgrading the entire app. In an app with mixed synchronous and concurrent roots, a unified commit would mean that every call to `Context.write` has to be synchronous, which probably makes this option a non-starter.

In either of these models, React would need to track a global list of all roots in order to schedule updates on them. This isn't something we do currently, and it means roots would no longer be automatically garbage collected; discarded roots would need to be explicitly unmounted to remove them from the global list.

What's clear from exploring this issue and others is that there are significant implementation costs to supporting multiple roots. In the future, we may move to a portal-first API that enforces a single root by default, and extract support for multiple roots to a separate package.

### Discordance with Hook API

`Context.write` has a similar API to the `useState` Hook, and `contextDidUpdate` is similar to `useEffect`. Perhaps we could consider an alternate proposal that uses these Hooks directly. See to the "Alternatives" section for an example.

# Alternatives

## Do nothing and leave the problem to user space

We've come this far without the need for an API like this. But the main reason we need to address this now is because of Suspense and concurrent rendering. Given how important this use case is for Suspense, doing nothing seems unlikely.

## Mount a "shell" component and cache using local state

Another way to move this to user space would be to have a shell component at the root of the app that initially renders with no children. On mount, it schedules a re-render to mount the children. Because the shell is already mounted, the children can cache values in the shell's local state.

This doesn't address the multiple root problem, nor does it work with React's server renderer, which does not have updates.

## Use Hooks

Leverage Hooks for updates and effects, instead of adding new APIs that do a subset of the same thing. Here's an example that logs whenever the theme changes:

```js
// This needs to be a new method because using `createContext` would be a
// breaking change. But the opaque return type is the same.
const Theme = React.createGlobalState(ref => {
  const [theme, setTheme] = useState('light');
  useEffect(
    () => {
      console.log('Theme changed: ' + theme);
    },
    [theme],
  );

  useImperativeMethods(ref, () => ({setTheme}));

  return theme;
});

export function useTheme() {
  return useContext(Theme);
}

export function toggleTheme() {
  Theme.current.setTheme(theme => {
    return theme === 'light' ? 'dark' : 'light';
  });
}
```

A more realistic example is a router that sets up a global `'popstate'` listener.

Leveraging Hooks would have several advantages:

- Smaller API surface area. (Excepting the need for a separate `createContext` and `createGlobalState` APIs, though we could unify them in a major release.)
- Allows you to move or reuse code between component state and global state with minimal changes.

(I thought about moving this to a separate proposal, but it doesn't seem sufficiently different from `Context.write` to merit its own document. The bulk of the proposal is the same.)

# Adoption strategy

TODO

# How we teach this

TODO
