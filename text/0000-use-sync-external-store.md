- Start Date: 2022-03-23
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

_Note: This RFC is closer to an "intent to ship" and is different than the
process we typically do because it is the result of years of research into
concurrency, Suspense, and server rendering. All of what is posted here was
designed and discussed over the last year in the React 18 Working Group
([available here](https://github.com/reactwg/react-18/discussions)) and iterated
on in public experimental releases since 2018. **We'd like to get one final
round of broad public feedback from the community before shipping in case there
are new concerns that have not been discussed before.** You can consider the
Working Group to be a part of the RFC, so please feel free to quote and discuss
any content from it when commenting on the RFC here._

# Summary

Introduce a new built-in hook API, useSyncExternalStore, for reading and
subscribing from external data sources in a way that's compatible with
concurrent rendering features like selective hydration and time slicing.

# Basic example

```js
import {useSyncExternalStore} from 'react';

const state = useSyncExternalStore(store.subscribe, store.getSnapshot);
```

Selecting a specific field using an inline `getSnapshot` function:

```js
const selectedField = useSyncExternalStore(
  store.subscribe,
  () => store.getSnapshot().selectedField,
);
```

Reading an initial snapshot during server rendering and hydration, to prevent
server-client mismatches:

```js
const selectedField = useSyncExternalStore(
  store.subscribe,
  () => store.getSnapshot().selectedField,
  () => INITIAL_SERVER_SNAPSHOT.selectedField,
);
```

getSnapshot is used to check if the subscribed value has changed since the
last time it was rendered, so the result needs to be referentially stable. That
means it either needs to be an immutable value like a string or number, or it
needs to be a cached/memoized object.

As a convenience, we will provide a version of the API with automatic support
for memoizing the result of getSnapshot:

```js
import {useSyncExternalStoreWithSelector} from 'use-sync-external-store/with-selector';

const selection = useSyncExternalStoreWithSelector(
  store.subscribe,
  store.getSnapshot,
  getServerSnapshot,
  selector,
  isEqual,
);
```

# Motivation

This proposal is an evolution of the RFC for
[useMutableSource](https://github.com/reactjs/rfcs/pull/147), which it replaces.

It has a few high level goals:

- Provide a first-class API to read and subscribe to external data sources.
  Subscriptions are an extremely common pattern in React applications, and many
  third-party library bundle their own implementation. This new built-in API
  will shift the implementation burden into React, reducing implementation
  complexity for developers.
- Ensure compatibility with concurrent rendering features, like Suspense and
  time slicing. Reading external state is tricky to get right in a concurrent
  environment. Mistakes can lead to data races and inconsistencies, which can
  lead to logical errors or visual tearing in the UI. Instead of leaving
  developers to deal with this complexity on their own, React will provide an
  official solution that's easy to use and is guaranteed to prevent concurrent
  data races.
- Prevent mismatches when hydrating server-rendered content. The initial render
  on the client needs to match exactly what was rendered on the server, even if
  the underlying data has changed in the meantime. useSyncExternalStore will
  read the correct version of data during hydration, then apply additional
  changes in a subsequent update.

# Detailed design

A flaw we discovered when testing useMutableSource (the previous iteration of
this proposal) is that it can sometimes cause visible parts of the UI to be
replaced with a fallback, even when the update is wrapped with startTransition
(the API that is designed to avoid this scenario).

The reason is that startTransition relies on the ability to maintain multiple
versions of a UI simultaneously ("concurrently"): the current UI that’s visible
on screen, and a work-in-progress UI that is prepared in the background while
data progressively streams in. React can do this with its built-in state APIs —
useState and useReducer — but not for state that lives outside React, because we
only can access a single version of state at a time. (To illustrate with an
example, Redux’s store has a getState method, but it doesn’t have a
getBackgroundState method; we could theoretically implement a contract to
support concurrent data stores, but that’s outside the scope of this proposal.)

Our original strategy was to provide partial support for concurrent features:
start rendering concurrently, and only deopt back to synchronous when we detect
an inconsistency. “Deopt" in this context can mean:

1. Disabling time-slicing and reverting to fully synchronous, blocking rendering
2. During a refresh transition, hiding the UI and replacing it with a fallback
   instead of waiting for the new data to load in the background.

We went through great pains to preserve time-slicing as much as possible (1),
even if it meant hiding already-visible UI with a fallback (2).

We’ve since concluded that this trade-off is backwards: Replacing visible
content with a fallback is a significant regression in the user experience,
especially if it happens unpredictably. By contrast, occasionally disabling
time-slicing — while not ideal — has a much less dramatic effect on the end
user experience.

Here's an example of a sequence that would lead to an unexpected fallback in the
useMutableSource API. Suppose you use external state to implement a
tab switcher:

- The user navigates to a new tab by mutating an external store.
- The loading state for the new tab hasn't been fetched yet, so React suspends
  the navigation transition.
- In the meantime, the user hovers over a tooltip, which triggers a high
  priority update.
- React interrupts the tab navigation and starts rendering the tooltip.
- The tooltip reads from the external store, which still has pending mutations.
- To prevent an inconsistently, React synchronously finishes processing the
  pending store mutations.
- Because the tab navigation is still suspended, React must replace the tab
  switcher with a fallback.

As we were brainstorming alternative strategies, a key revelation was that if
you can’t rely on startTransition to always avoid bad fallbacks caused by a
store update, then you shouldn’t ever rely on it — you should avoid the
fallbacks in some other way. For example, you could do it the same way you would
today without Suspense: by waiting for new data to load before triggering the
update.

The next key revelation was that we can avoid deopts during updates triggered by
React state transitions if updates triggered by external stores are always
synchronous. That’s because if updates to stores are synchronous, they are
guaranteed to be consistent.

So, in this proposal:

- Updates triggered by a store change will always be synchronous, even when
  wrapped in startTransition
- In exchange, updates triggered by built-in React state will never deopt by
  showing a bad fallback, even if it reads from a store during the same render

# Drawbacks

- As discussed above, to prevent data races, you can't trigger a transition using
  a store update — updates to the store always render synchronously. This isn't
  the same as saying all data access must be synchronous — you can still read from
  the store during a concurrent transition if the transition was triggered by a
  React state update, like useState or useReducer. But this does mean that
  useSyncExternalStore is not the ideal API for implementing things like route
  transitions, though we're exploring additional APIs that will enable this use
  case. (See "Unresolved questions" for more.)
- getSnapshot must return a cached value. That is, if getSnapshot is called
  multiple times in a row, it must return the same exact value unless there was
  a store update in between. Some existing store implementations don't follow
  this pattern because they create new objects on demand every time the store is
  read. To make the API easier to adopt, we provide a
  useSyncExternalStoreWithSelector API that automatically caches the
  last result.

# Alternatives

- Leave the problem to be solved in userspace. While it’s possible to implement
  a concurrent-compatible subscription in userspace, it’s very tricky to get
  right in all the edge cases. It's also impractical to expect userspace
  implementations to anticipate future React features that we may add later on.
- Ship useMutableSource instead.

# Adoption strategy

Our goal is for all subscription-based libraries to migrate their
implementations to useSyncExternalStore.

To encourage adoption by open source libraries, we will provide a shim that is
compatible with older versions of React.

```js
import {useSyncExternalStore} from 'use-sync-external-store/shim';
```

The shim will prefer the built-in API when it is available, so that users get
the correct implementation regardless of which version of React they’re running.

We are also considering a heuristic to detect when a userspace store update is
wrapped with startTransition, so we can print advice to the console (in
development mode) to use useSyncExternalStore instead. The heuristic is to count
how many separate components are updated within a single startTransition call.
If it’s greater than some arbitrary threshold, say 20, we can infer that it
likely contains a subscription.

# How we teach this

This API is designed to be used primarily by library and framework authors, not
by the typical React developer. However, the API is simple enough that we
expect most users can pick it up with minimal effort by reading
the documentation.

# Unresolved questions

Although updates triggered through useSyncExternalStore are always synchronous,
we intend to add another API to support concurrent data access, too. It will
almost certainly have a different design and set of trade offs from
useSyncExternalStore, so it won't replace the need for this API.

We're not ready to share a proposal for concurrent stores yet, but we'll provide
more details in the weeks after React 18.0 is released.

In the meantime, useSyncExternalStore is the recommended solution for all
external data subscriptions. Use built-in React state (useState, useReducer) for
concurrent transitions, like navigating between pages.
