# RFC: Unified cancellation via AbortSignal across React async APIs

- Start Date: 2025-10-23
- RFC PR: 
- React Issue: 

## Summary

Introduce an opt-in, web-standard cancellation model using `AbortSignal` that flows consistently through React async features: `startTransition`, Suspense with `use`, and server actions. The goal is to stop stale work predictably on both client and server, reduce wasted computation and I/O, and align React with platform semantics. All cancellation is advisory and backwards compatible—existing code continues to work unchanged.

## Motivation

React now spans client and server execution with transitions, Suspense, `use`, and server actions. Today, when a user navigates away or starts a superseding transition, previously initiated work often continues to run. Examples:

- A transition that becomes obsolete continues resolving data and scheduling renders.
- A promise read via `use` keeps resolving after its Suspense boundary is abandoned.
- A server action keeps running even if the user leaves the page, causing unnecessary server load.

Lack of a shared cancellation model makes performance tuning and correctness harder. A unified `AbortSignal` based approach would provide a predictable, composable way to stop irrelevant work and free resources early.

## Detailed design

This RFC proposes a minimal cross-cutting integration that threads `AbortSignal` through core async APIs. The design is opt-in and advisory: React remains free to commit already-completed urgent work, but ongoing async work observes cancellation consistently.

### Design principles

All cancellation is optional and advisory. Existing code works unchanged. `startTransition` can still be called without expecting a return value. Signals inform but don't control React's scheduling—React may commit already-prepared updates even after cancellation. Signals can be chained and passed through async boundaries, allowing userland code to participate. The design uses Web Platform `AbortSignal` and `AbortController`, not custom primitives.

### 1) Transitions

```ts
type TransitionOptions = {
  signal?: AbortSignal;
};

type TransitionHandle = {
  signal: AbortSignal;
  cancel: () => void;
};

// Overloads to maintain backwards compatibility
declare function startTransition(
  fn: (signal: AbortSignal) => void
): TransitionHandle;

declare function startTransition(
  fn: (signal: AbortSignal) => void,
  options?: TransitionOptions
): TransitionHandle;

// useTransition hook returns startTransition with same signature
declare function useTransition(): [
  isPending: boolean,
  startTransition: (
    fn: (signal: AbortSignal) => void,
    options?: TransitionOptions
  ) => TransitionHandle
];
```

`startTransition` now returns a `TransitionHandle` containing `{ signal, cancel }`. The callback receives the `signal` as its first parameter, enabling immediate access. React aborts `signal` automatically when the transition is superseded by a newer transition or when the associated UI path is abandoned (component unmounts, navigation occurs). Users may call `cancel()` to abort explicitly at any time. 

If an external `signal` is provided via `options.signal`, React will listen to it and cancel the transition if that external signal aborts. When `options.signal` aborts, React aborts the transition's internal signal and the returned handle's `signal`. Aborting the handle also aborts the internal signal but does not change the external signal's state. If `options.signal` is already aborted at call time, React still invokes the callback synchronously and returns a handle whose `signal` is immediately aborted with the same `reason`.

Any async work within the transition that accepts a signal should observe `signal.aborted === true` and stop gracefully.

For backwards compatibility, callbacks that ignore parameters (e.g., `startTransition(() => {...})`) remain valid; React still returns a handle, and TypeScript accepts a zero-param callback where a one-param callback is expected. Existing calls to `startTransition(fn)` that don't expect a return value continue to work.

### 2) Suspense and `use`

```ts
// React internally provides a signal when rendering within a Suspense boundary
declare function use<T>(
  resource: Promise<T> | { read: () => T }
): T;

// Hook to access the current boundary's signal
// Returns the nearest Suspense boundary's signal
// Throws in development, returns inert signal in production if called outside boundary
declare function useSuspenseSignal(): AbortSignal;
```

React automatically creates and manages an `AbortSignal` for each Suspense boundary. When a Suspense boundary unmounts or becomes hidden (offscreen), React aborts its signal. The `useSuspenseSignal()` hook allows components to access the nearest ancestor Suspense boundary's signal. If called outside a Suspense boundary, the hook throws in development and returns a permanently-aborted inert signal in production. This mirrors other dev-only safety checks while keeping production code robust. When Suspense boundaries are nested, the hook returns the signal from the closest boundary. Promises passed to `use()` should be created with this signal to enable cancellation:

```ts
function MyComponent() {
  const signal = useSuspenseSignal();
  const data = use(fetch('/api/data', { signal }).then(r => r.json()));
  return <div>{data.name}</div>;
}
```

For promises created outside the component (e.g., in a cache or loader), the promise factory pattern enables signal threading:

```ts
// Cache or loader layer
function createDataLoader(id: string, signal: AbortSignal) {
  return fetch(`/api/data/${id}`, { signal }).then(r => r.json());
}

// Component
function MyComponent({ id }) {
  const signal = useSuspenseSignal();
  const data = use(createDataLoader(id, signal));
  return <div>{data.name}</div>;
}
```

The lifecycle proceeds as follows: component renders within a Suspense boundary, `useSuspenseSignal()` returns the boundary's active signal, promise is created with that signal, and if the boundary unmounts or becomes offscreen the signal aborts. Fetch and other platform APIs automatically cancel ongoing work.

### 3) Server actions

```ts
// Server action with optional context parameter
export async function myAction(
  formData: FormData,
  context?: { signal?: AbortSignal }
) {
  // Pass context.signal to fetch and other cancellable work
  const signal = context?.signal;
  if (signal) {
    const res = await fetch('https://api.example.com/data', { signal });
    return res.json();
  }
}
```

When a server action is invoked from within a transition or form submission, React automatically propagates the associated abort semantics to the server runtime, which creates a corresponding `AbortSignal` for the action's execution. The `AbortSignal` object itself is not serialized across the network; the server runtime reflects client aborts into a server-side signal that behaves equivalently. The signal is passed as an optional second parameter via a context object. If the client navigates away, supersedes the transition, or cancels explicitly, the signal is aborted. The server receives notification of the abort through its signal, which propagates to `fetch` and other cancellable I/O.

For streaming RSC (React Server Components) connections, the abort signal is communicated via the existing bidirectional channel. For traditional HTTP POST actions, the client closes the connection or sends an abort message if supported. The server action runtime provides the context parameter with the signal state.

The context parameter is optional and defaults to `undefined`. Existing server actions that don't accept a second parameter continue to work unchanged. Server actions can check for `context?.signal` to opt into cancellation support.

### 4) DevTools and tracing

DevTools Profiler can display cancelled work items with metadata showing which transition, Suspense boundary, or server action was cancelled, the cancellation reason from `signal.reason` (such as "superseded", "navigation", or "unmount"), when cancellation occurred, and may surface counts of cancelled requests and skipped renders where instrumentation is available. Values are approximate and diagnostic only. Cancelled transitions appear with a distinctive marker in the timeline. A "Cancellation" tab shows all aborted signals during the profiling session.

Example timeline display:

```
Timeline:
  ├─ Transition #1 [CANCELLED] (superseded by Transition #2)
  │  ├─ Fetch: /api/search?q=abc [CANCELLED]
  │  └─ Render: <SearchResults> [STOPPED]
  └─ Transition #2 [COMPLETED]
     └─ Fetch: /api/search?q=abcd [SUCCESS]
```

This is diagnostic only. No application behavior depends on DevTools being present.

### Scheduling and guarantees

Cancellation is advisory and best-effort. React observes signals but maintains control over commit decisions. If an update is already committed to the DOM, React will not roll it back due to an abort. React continues to coalesce updates and preserve scheduling semantics. The `signal` allows userland code and platform I/O (fetch, streams) to stop ongoing work.

When React aborts a signal it sets `signal.reason` to a React-defined value (e.g., `"superseded"`, `"navigation"`, or `"unmount"`). Applications may branch on `signal.reason` for diagnostics or logging; behavior must not depend on undocumented reason strings as these may change.

Ordering: React first marks the transition/boundary signal as aborted (microtask), then prevents further work tied to that signal from being scheduled. Work already committed is not rolled back; pending effects created by the aborted render do not run.

If `cancel()` is called while React is committing, the commit completes normally. Subsequent renders in the same transition will observe the aborted state. Async work checking `signal.aborted` will stop at their next checkpoint.

When a transition runs within a Suspense boundary, both signals coexist. If either signal aborts, the work should stop. Userland code can use `AbortSignal.any([signal1, signal2])` (proposed platform API) or similar patterns to compose signals.

## Examples

### Example 1: Search box with superseding transitions

```tsx
function SearchBox() {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState([]);
  const handleRef = useRef<TransitionHandle | null>(null);

  useEffect(() => {
    if (!query) {
      setResults([]);
      return;
    }

    // Cancel previous search if still running
    handleRef.current?.cancel();

    // Start new transition with automatic cancellation
    // Note: cancellation replaces typical debouncing;
    // you can still debounce setQuery if desired.
    handleRef.current = startTransition((signal) => {
      fetch(`/api/search?q=${encodeURIComponent(query)}`, { signal })
        .then(res => res.json())
        .then(data => {
          // Only update if not cancelled
          if (!signal.aborted) {
            setResults(data.results);
          }
        })
        .catch(err => {
          // AbortError is expected on cancellation
          if (err?.name !== 'AbortError') {
            console.error('Search failed:', err);
          }
        });
    });

    // Cleanup: cancel on unmount or query change
    return () => handleRef.current?.cancel();
  }, [query]);

  return (
    <div>
      <input 
        value={query} 
        onChange={e => setQuery(e.target.value)} 
        placeholder="Search..."
      />
      <ul>
        {results.map(r => <li key={r.id}>{r.title}</li>)}
      </ul>
    </div>
  );
}
```

### Example 2: Suspense + `use` with cancellation

```tsx
function UserProfile({ userId }: { userId: string }) {
  return (
    <Suspense fallback={<Spinner />}>
      <UserDetails userId={userId} />
    </Suspense>
  );
}

function UserDetails({ userId }: { userId: string }) {
  // Access the Suspense boundary's signal
  const signal = useSuspenseSignal();
  
  // Create promise with signal - will auto-cancel if boundary unmounts
  const userPromise = fetch(`/api/user/${userId}`, { signal })
    .then(r => r.json());
  
  const user = use(userPromise);
  
  return (
    <div>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
    </div>
  );
}
```

### Example 3: Server action with cancellation

```tsx
// app/actions.ts (server)
'use server';

export async function savePost(
  formData: FormData,
  context?: { signal?: AbortSignal }
) {
  const signal = context?.signal;
  const title = formData.get('title') as string;
  const content = formData.get('content') as string;

  // Pass signal to external API calls
  const res = await fetch('https://api.example.com/posts', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ title, content }),
    signal, // Automatically cancels if client aborts
  });

  if (!res.ok) throw new Error('Failed to save');
  return res.json();
}

// app/components/PostForm.tsx (client)
'use client';

import { savePost } from '../actions';

function PostForm() {
  const [isPending, startTransition] = useTransition();

  const handleSubmit = (formData: FormData) => {
    startTransition(async (signal) => {
      try {
        const result = await savePost(formData, { signal });
        console.log('Saved:', result);
      } catch (err) {
        if (err?.name !== 'AbortError') {
          console.error('Save failed:', err);
        }
      }
    });
  };

  return (
    <form action={handleSubmit}>
      <input name="title" placeholder="Title" required />
      <textarea name="content" placeholder="Content" required />
      <button type="submit" disabled={isPending}>
        {isPending ? 'Saving...' : 'Save Post'}
      </button>
    </form>
  );
}
```

### Example 4: Nested signals with composition

```tsx
function DashboardPanel() {
  return (
    <Suspense fallback={<Loading />}>
      <DashboardData />
    </Suspense>
  );
}

function DashboardData() {
  const suspenseSignal = useSuspenseSignal();
  const [data, setData] = useState(null);

  useEffect(() => {
    // Start a transition within a Suspense boundary
    const handle = startTransition((transitionSignal) => {
      // Compose both signals - abort if either cancels
      // Note: AbortSignal.any() is a newer platform API
      // For older browsers, use a polyfill or custom composition
      const combinedSignal = 'any' in AbortSignal
        ? AbortSignal.any([suspenseSignal, transitionSignal])
        : composeSignals([suspenseSignal, transitionSignal]);

      fetch('/api/dashboard', { signal: combinedSignal })
        .then(r => r.json())
        .then(data => {
          if (!combinedSignal.aborted) {
            setData(data);
          }
        })
        .catch(err => {
          if (err?.name !== 'AbortError') {
            console.error(err);
          }
        });
    });

    return () => handle.cancel();
  }, [suspenseSignal]);

  return <div>{data ? <Dashboard data={data} /> : <Loading />}</div>;
}
```

### Example 5: Error handling patterns

```tsx
function DataFetcher({ endpoint }: { endpoint: string }) {
  const [data, setData] = useState(null);
  const [error, setError] = useState(null);

  useEffect(() => {
    const handle = startTransition((signal) => {
      setError(null);
      
      fetch(endpoint, { signal })
        .then(res => {
          if (!res.ok) throw new Error(`HTTP ${res.status}`);
          return res.json();
        })
        .then(data => {
          // Check signal before updating state
          if (!signal.aborted) {
            setData(data);
          }
        })
        .catch(err => {
          // Don't treat cancellation as an error
          if (err?.name === 'AbortError') {
            console.log('Fetch cancelled');
          } else if (!signal.aborted) {
            setError(err?.message || 'Unknown error');
          }
        });
    });

    return () => handle.cancel();
  }, [endpoint]);

  if (error) return <div>Error: {error}</div>;
  if (!data) return <div>Loading...</div>;
  return <div>Data: {JSON.stringify(data)}</div>;
}
```

### Helper: Signal composition for older browsers

For browsers that don't yet support `AbortSignal.any()`, a simple composition helper. React does not depend on `AbortSignal.any()`. The helper shown here is sufficient for broad browser support.

```ts
function composeSignals(signals: AbortSignal[]): AbortSignal {
  const controller = new AbortController();
  
  for (const signal of signals) {
    if (signal.aborted) {
      controller.abort();
      break;
    }
    signal.addEventListener('abort', () => controller.abort(), { once: true });
  }
  
  return controller.signal;
}
```

## How we teach this

### Core concepts

React uses the existing Web Platform `AbortController` and `AbortSignal` APIs, not a custom cancellation system. Developers already familiar with fetch cancellation will recognize the pattern. Cancellation is entirely optional—existing code works without changes. Teams can adopt incrementally, starting with high-impact areas like search or data fetching.

React provides signals to inform code that work is no longer needed. React won't roll back committed UI. Code decides how to respond to signals. `AbortSignal` acts as a notification: React indicates work is no longer needed, code checks whether to continue, and platform APIs (fetch, streams) automatically respect the signal.

### Documentation structure

Documentation should include an introduction guide covering why cancellation matters (performance, UX), when to use it (searches, transitions, data loading), and a quick start with `startTransition` example. The API reference documents `startTransition` with signal parameter and return value, the `useSuspenseSignal()` hook, server action context parameter, and the DevTools cancellation tab. A patterns cookbook shows search with debouncing and cancellation, infinite scroll with cancellable fetches, form submission with server action cancellation, router navigation with transition cancellation, and composing multiple signals. Finally, a migration guide covers identifying opportunities for cancellation in existing code, adding signals to custom data fetching libraries, testing cancellation behavior, and common pitfalls and how to avoid them.

### Common Pitfalls to Highlight

- **Don't create controllers in render**: Use refs, memos, or React-managed signals
- **Always check `signal.aborted` before state updates**: Prevents updates after cancellation
- **Handle `AbortError` gracefully**: It's expected, not an error condition
- **Don't ignore the signal**: Even if cancellation is advisory, respecting it improves performance
- **Strict Mode double-invoke**: In Strict Mode, React may invoke render/teardown twice in development. Ensure cancellation paths are idempotent (aborting an already-aborted controller is a no-op)

### Teaching progression

Beginners start with `startTransition` and signals for simple fetch cancellation. Intermediate users thread signals through data loading layers and caches. Advanced usage includes composing multiple signals, integrating with routers, and optimizing server actions.

## Drawbacks

The proposal increases API surface: `startTransition` now returns a value and accepts a signal parameter, a new `useSuspenseSignal()` hook is introduced, server actions gain an optional context parameter, and developers need to learn more concepts.

Ecosystem adoption is required. Data fetching libraries need updates to thread signals through their APIs, state management libraries should integrate with transition signals, and partial adoption means inconsistent cancellation behavior across apps.

There is potential for misuse. Developers might ignore signals (defeating the purpose), incorrectly composed signals could cancel work prematurely, memory leaks can occur if controllers aren't properly cleaned up, and confusion may arise between "advisory" cancellation and mandatory cancellation.

Implementation complexity increases. React internals need to track and manage signals per transition/boundary, server-client signal propagation adds protocol complexity, and DevTools integration requires new UI and instrumentation.

Minor breaking changes are possible. The `startTransition` signature changes (though backwards compatible via overloads), server action signature changes (optional parameter maintains compatibility), and existing code using the return value of `startTransition` (currently `undefined`) may break.

Testing becomes more challenging. Tests need to account for cancellation timing, race conditions become more prevalent, and mocking `AbortSignal` in tests adds complexity.

## Alternatives

### 1. Status Quo

Continue without built-in cancellation support. Leave it to userland libraries and frameworks. This avoids API changes and additional complexity in React core, leaving developers with full control. However, it leads to inconsistent patterns across the ecosystem with no unified story for transitions, Suspense, and server actions. Every library/framework reinvents cancellation differently, making it harder to optimize for React's internals.

### 2. Userland-Only Solutions

Rely on data fetching libraries (TanStack Query, SWR, Apollo) to implement cancellation. Libraries can tailor cancellation to their specific needs without any changes to React APIs—a proven approach already used successfully. However, this doesn't cover all async work (only library-managed fetches), lacks integration with React's scheduling and transitions, leaves server actions uncancellable, and results in a fragmented developer experience.

### 3. Custom Cancellation Token API

Create a React-specific cancellation API instead of using `AbortSignal`. For example:

```ts
type CancellationToken = {
  cancelled: boolean;
  onCancel: (callback: () => void) => void;
};
```

This could be tailored to React's specific needs and might be simpler than `AbortController`/`AbortSignal`. However, it adds a new primitive instead of using platform standards, doesn't integrate with fetch and other platform APIs, causes ecosystem fragmentation with two cancellation systems, and goes against React's philosophy of aligning with the platform.

### 4. Automatic Cancellation Without Explicit API

React automatically cancels all ongoing work when transitions are superseded, without exposing signals. This offers a simple developer experience with no new API surface and no opt-in required. However, developers would have no way to customize cancellation behavior, can't thread signals to custom async work, have no explicit cancel() method for manual control, and black box behavior is harder to debug.

### 5. React-Specific Hooks Only (No Platform Integration)

Provide `useTransitionSignal()` and `useSuspenseSignal()` but don't integrate with platform `AbortSignal`. This simplifies implementation with no platform API dependencies. However, it doesn't compose with fetch and other platform APIs, requires custom integration code for every async API, and doesn't leverage the platform's built-in cancellation infrastructure.

This RFC chooses `AbortSignal` integration because it leverages existing platform standards rather than inventing new primitives, works with fetch and other cancellable platform APIs, is already supported by many libraries, allows signals to be chained and combined, and provides a clear API surface that makes behavior predictable and debuggable.

## Prior art

### Web Platform Standards

The foundation of this proposal is the WHATWG standard `AbortController` and `AbortSignal`:

- **WHATWG DOM Standard**: AbortController and AbortSignal — https://dom.spec.whatwg.org/#aborting-ongoing-activities
  - Defines the standard cancellation primitive used across web APIs
  - Widely implemented in all modern browsers
  
- **Fetch Standard**: Integration of `signal` with fetch() — https://fetch.spec.whatwg.org/
  - Standard pattern for cancellable HTTP requests
  - Model for how signals compose with async operations

- **MDN Documentation**: AbortController overview — https://developer.mozilla.org/en-US/docs/Web/API/AbortController
  - Comprehensive developer guide with examples
  - Shows established patterns familiar to React developers

- **`AbortSignal.any()` Proposal**: Composing multiple signals — https://github.com/whatwg/dom/issues/920
  - Platform feature that enables the signal composition patterns in Example 4
  - Shows forward-thinking alignment with platform evolution

### React Ecosystem

This proposal builds on React's concurrent features and addresses community needs:

- **React 19 Features**: Transitions, `use()`, and server actions — https://react.dev/blog/2024/04/25/react-19
  - Context for the async APIs this RFC enhances
  - Foundation for cancellation integration

- **Community Requests**: Multiple GitHub issues demonstrate demand:
  - AbortSignal integration requests — https://github.com/facebook/react/issues?q=is%3Aissue+AbortSignal
  - Server action cancellation needs — https://github.com/facebook/react/issues?q=is%3Aissue+server+actions+cancel
  - Transition lifecycle and async patterns — https://github.com/facebook/react/issues?q=is%3Aissue+startTransition+await

### Data Fetching Libraries

Major React data libraries already support `AbortSignal`, demonstrating ecosystem readiness:

- **TanStack Query (React Query)**: Supports `signal` in query functions — https://tanstack.com/query/latest/docs/react/guides/query-cancellation
  - Pattern: `queryFn: ({ signal }) => fetch(url, { signal })`
  - Proven approach with millions of downloads

- **SWR**: Experimental signal support in fetchers — https://swr.vercel.app/docs/advanced/understanding
  - Community interest in better React integration
  
- **Apollo Client**: Custom cancellation via observables
  - Could benefit from standardized `AbortSignal` pattern

### React Frameworks

Next.js and Remix provide context for server-side integration:

- **Next.js App Router**: Server components and actions — https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations
  - Could integrate cancellation in loader and action patterns
  
- **Remix**: Action and loader cancellation patterns — https://remix.run/docs/en/main/guides/data-loading
  - Already implements navigation cancellation for loaders

### Other Framework Patterns

Cancellation patterns from other JavaScript frameworks inform the design:

- **Angular**: RxJS observables with `unsubscribe()` for automatic cleanup
- **Vue 3**: Composition API lifecycle hooks for resource cleanup
- **Solid.js**: Reactive scope disposal with automatic resource cleanup

### Cross-Language Inspiration

Cancellation is a solved problem in other languages, providing design guidance:

- **Go**: `context.Context` propagates cancellation through call chains
- **C#**: `CancellationToken` for coordinated async cancellation
- **Kotlin Coroutines**: Structured concurrency with parent-child cancellation
- **Swift Concurrency**: Task cancellation with cooperative checking

### RFC Process

- **React RFC Repository**: Process and template — https://github.com/reactjs/rfcs
  - Establishes the review and consensus process for this proposal

## Unresolved questions

### API Design

1. **`startTransition` Return Value**:
   - Should it always return a `TransitionHandle`, or only when the callback accepts a parameter?
   - How do we maintain backwards compatibility for code that doesn't expect a return value?
   - Should there be separate `startCancellableTransition()` API?

2. **`useTransition` Hook Integration**:
   - Should `useTransition()` also provide access to the signal?
   - Pattern: `const [isPending, startTransition, signal] = useTransition()`?
   - Or: `const [isPending, startTransition, handle] = useTransition()`?

3. **Signal Lifetime and Garbage Collection**:
   - When are transition signals GC'd if the handle is retained?
   - Should there be explicit disposal beyond abort?
   - Memory implications for long-lived applications with many transitions

4. **`signal.reason` Values**:
   - Should React document the complete list of reason strings?
   - Can userland code set custom reasons when calling cancel() explicitly?
   - Should reasons be typed (e.g., via TypeScript union types)?

### React Features Integration

5. **`useOptimistic` Interaction**:
   - How does optimistic state interact with cancelled transitions?
   - Should optimistic updates roll back when signals abort?
   - Race condition handling between optimistic updates and cancellation

6. **Concurrent Rendering**:
   - How do multiple concurrent transitions with different priorities interact?
   - Does higher priority work cancel lower priority work automatically?
   - Should signals reflect priority relationships?

7. **Server Components**:
   - How do RSC async components receive cancellation signals?
   - Can an RSC tree be cancelled mid-stream?
   - Interaction with streaming SSR and selective hydration

8. **Partial Pre-rendering (PPR)**:
   - How does cancellation work with PPR's static/dynamic boundaries?
   - Can dynamic segments be cancelled if navigation occurs during streaming?

### Router and Framework Integration

9. **Router Navigation**:
   - Should routers automatically cancel on navigation?
   - How do framework routers (Next.js App Router, Remix) integrate?
   - Should there be a standard router cancellation protocol?

10. **Nested Routes**:
    - When a parent route changes, should all child route transitions cancel?
    - How to handle layout persistence with cancellation?

11. **History Navigation**:
    - Back/forward navigation cancellation semantics
    - Should pending transitions cancel on history changes?

### DevTools and Debugging

12. **DevTools Telemetry**:
    - What minimal data should be surfaced by default?
    - Privacy implications of recording cancelled work
    - Performance overhead of instrumentation

13. **Source Maps and Stack Traces**:
    - How to show where cancellation originated?
    - Stack trace preservation for cancelled promises

### Platform and Browser

14. **`AbortSignal.any()` Availability**:
    - What's the polyfill story for older browsers?
    - Should React provide a built-in utility for composing signals (see helper in Examples section)?
    - Browser support is currently limited; should docs recommend the fallback pattern?

15. **Browser Tab Backgrounding**:
    - Should transitions cancel when tabs go to background?
    - Integration with Page Visibility API

16. **Service Workers**:
    - How do signals propagate to/from service workers?
    - Offline scenarios and cancellation

### Testing and Development

17. **Testing Utilities**:
    - Should React Testing Library expose cancellation helpers?
    - How to test cancellation timing reliably?
    - Mock signal APIs for unit tests

18. **Strict Mode**:
    - Should Strict Mode double-cancel to catch bugs?
    - Development-only cancellation warnings

### Edge Cases

19. **Signal Already Aborted**:
    - Behavior is now specified in the Transitions section
    - React invokes callback synchronously and returns immediately-aborted handle with same reason

20. **Circular Signal Dependencies**:
    - Can signal composition create cycles?
    - How to detect and handle this?

21. **Memory Leaks**:
    - Best practices for avoiding leaked controllers
    - Should React warn about common leak patterns?

## Migration path

### For Application Developers

**Phase 1: Awareness (No Changes Required)**
- All existing code continues to work unchanged
- `startTransition()` can still be called without expecting a return value
- Server actions without context parameters work as before

**Phase 2: Gradual Adoption (Opt-in)**

Identify high-impact areas like search, infinite scroll, and form submissions. Update one feature at a time, starting with a single search box or data fetching component. Test cancellation behavior to verify cancelled requests don't update state. Monitor DevTools using the Profiler to see cancelled work and savings.

**Phase 3: Ecosystem Integration**

Update the data fetching layer to accept and forward signals. Integrate with the router for navigation cancellation. Add signals to server actions that perform expensive operations.

### For Library Authors

**Data Fetching Libraries (TanStack Query, SWR, Apollo)**:
```ts
// Before
function useQuery(key, fetcher) {
  // ... existing implementation
}

// After (backwards compatible)
function useQuery(key, fetcher, options?: { signal?: AbortSignal }) {
  const signal = options?.signal;
  // Thread signal through to fetcher
  // Cancel when component unmounts or signal aborts
}
```

**State Management Libraries**:
- Integrate with transition signals for async actions
- Cancel pending middleware/effects when signals abort
- Provide utilities for composing signals

**Router Libraries**:
- Automatically create and cancel signals on navigation
- Expose signals to loader functions
- Cancel previous route transitions when navigating

### For Framework Authors (Next.js, Remix, etc.)

Framework authors should integrate routers by wrapping navigation in `startTransition` with signals, passing signals to route loaders and actions, and cancelling on navigation or back/forward events. Server action runtimes need to implement signal propagation from client to server, provide context parameters to action functions, and handle aborted signals gracefully. Build-time optimizations can analyze which components use cancellation, generate code that threads signals automatically, and warn about missing signal handling in critical paths.

### Backwards compatibility

The following continue to work without modification:
- Existing `startTransition(() => {...})` calls
- Server actions without context parameter
- Components that don't use `useSuspenseSignal()`
- All current React patterns and APIs

Potential issues:
- Code that relies on `startTransition` returning `undefined` (very rare)
- Custom polyfills of React APIs may need updates
- Tests that mock React may need adjustments

### Codemod opportunities

Potential codemods could assist migration by adding signal parameters to fetchers:

```ts
// Before
const data = await fetch(url).then(r => r.json());

// After
const data = await fetch(url, { signal }).then(r => r.json());
```

Another codemod could convert manual cancellation to React signals:

```ts
// Before
const controller = new AbortController();
useEffect(() => () => controller.abort(), []);

// After
const signal = useSuspenseSignal();
// Use signal directly, no manual cleanup needed
```

## Adoption strategy

### Rollout plan

The React core team ships behind an experimental flag initially, gathers feedback from framework authors, iterates on the API based on real-world usage, and promotes to stable once patterns are proven.

Documentation includes a new "Cancellation" section in the main docs, updates to `startTransition`, `use`, and server actions docs, comprehensive examples and patterns guide, and a DevTools guide for the cancellation tab.

Ecosystem communication involves publishing this RFC and gathering community feedback, working with major library authors for early adoption, creating migration guides for common libraries, and hosting workshops and talks on cancellation patterns.

### Incremental adoption

Teams can start small with a single feature (e.g., search box), measure impact using DevTools to quantify saved work, and expand gradually to more features over time. There is no pressure to adopt everything at once.

For the ecosystem, core libraries (TanStack Query, SWR, React Router) adopt first, followed by framework integration (Next.js, Remix), then community libraries as benefits become clear.

### Success Metrics

- Percentage of transitions using cancellation
- Reduction in wasted network requests
- Improvement in app responsiveness
- DevTools adoption for cancellation debugging
- Library ecosystem adoption rate

### Support and Resources

- Dedicated docs section with interactive examples
- Troubleshooting guide for common issues
- Community Discord/forum for questions
- Regular office hours for library authors
- Blog posts and conference talks

## Performance considerations

### Memory overhead

Per transition overhead includes one `AbortController` instance (~200 bytes), one `AbortSignal` instance (~100 bytes), and internal React bookkeeping (~50-100 bytes), totaling ~300-400 bytes per transition. Per Suspense boundary overhead is similar, with lifetime tied to boundary mount/unmount, resulting in minimal impact for typical applications.

Controllers are GC'd when transitions complete. Signals are lightweight compared to prevented work. DevTools can help identify long-lived controllers.

### Runtime overhead

Signal checking: `signal.aborted` check is O(1) (~1-2 CPU cycles), negligible compared to fetch or render work, and only checked at cancellation checkpoints. Event listeners add minimal overhead—`signal.addEventListener('abort', ...)` is platform-optimized in modern browsers, with typical applications having 10-100 active listeners. Signal management in React internals adds <1% to transition overhead, primarily through boolean flag checks, with no impact when cancellation isn't used.

### Performance benefits

Network benefits include cancelled `fetch()` requests freeing browser connections, reduced bandwidth usage (especially on mobile), and lower server load from terminated requests. Typical savings: 20-50% of redundant requests eliminated.

CPU benefits include stopped render work freeing the main thread, prevented state updates reducing reconciliation, and cancelled promises not allocating result objects. Typical savings: 10-30% reduction in wasted CPU cycles.

Memory benefits include cancelled work releasing memory earlier, reduced pressure on the garbage collector, and fewer retained closures and promise chains. Net benefit is positive despite controller overhead.

### Benchmarks (Hypothetical)

```
Scenario: Search with 10 rapid keystrokes

Without Cancellation:
- 10 fetch requests (all complete): 500ms total
- 10 render cycles: 200ms total
- Wasted work: ~600ms (only last result matters)

With Cancellation:
- 10 fetch requests (9 cancelled): 50ms total
- 1 render cycle: 20ms total
- Total time: 70ms
- Improvement: ~8.5x faster
```

### When cancellation helps most

Cancellation is most beneficial for high-frequency user input (search, autocomplete), navigation-heavy apps (dashboards, admin panels), mobile networks (high latency, limited bandwidth), expensive operations (large data fetches, image processing), and server actions (reducing server load and costs).

Cancellation overhead exceeds benefits for very fast operations (<10ms) where cancellation checking adds proportionally more time, single-use transitions that never get cancelled, and static content that doesn't change based on user input. Use cancellation for user-interactive features, skip for one-time operations.

## Future work

Short-term enhancements could include router integration helpers (`useRouterSignal()` hook for navigation cancellation, automatic signal threading through route loaders, standard patterns for Next.js App Router and Remix), composition utilities (`React.composeSignals()` helper if `AbortSignal.any()` isn't available, `useComposedSignal()` hook, signal debugging utilities), and DevTools enhancements (flame graph showing cancelled work, cost analysis, suggestions for where to add cancellation).

Medium-term explorations might involve automatic cancellation heuristics (React automatically cancelling very low-priority work, heuristics for predicting work that will be superseded, adaptive cancellation based on device capabilities), streaming and incremental rendering (cancel partial RSC streams mid-flight, abort server-side rendering for offscreen content, progressive enhancement), and advanced Suspense integration (preload with signal, boundaries that auto-cancel on priority changes, cancellation budgets).

Long-term vision includes React Compiler integration (automatically insert signal checks in generated code, optimize away unnecessary signal propagation, static analysis of opportunities), cross-tab coordination (cancel work in background tabs, coordinate signals across SharedWorkers, distributed cancellation), platform proposals (work with WHATWG on extended `AbortSignal` capabilities, propose standard patterns for framework cancellation, advocate for better browser DevTools), and framework ecosystem work (standard cancellation protocol across React frameworks, shared utilities package for signal composition, best practices guide maintained by React team and community).
