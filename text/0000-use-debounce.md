- Start Date: (2025-09-14)
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

We propose adding an official useDebounce API to React. This hook delivers the
most commonly needed debounce behavior in a one-liner that is reliable,
concurrent-safe, and easy to teach. Instead of relying on third-party utilities,
React would provide a first-class, correct, and ergonomic solution for this
everyday use case.

# Basic example

```jsx
function SearchBox() {
  const [query, setQuery] = useState("");
  const debouncedQuery = useDebounce(query, 300);

  useEffect(() => {
    if (debouncedQuery) {
      fetch(`/api/search?q=${encodeURIComponent(debouncedQuery)}`);
    }
  }, [debouncedQuery]);

  return <input value={query} onChange={(e) => setQuery(e.target.value)} />;
}
```

With just one line, useDebounce ensures we avoid spamming the server while the
user types, yet always fetch the latest intended query.

# Motivation

Debouncing is one of the most common patterns in modern UIs:

- search boxes that should wait until typing pauses before firing a request,
- resize and scroll listeners that should not flood the browser with updates,
- expensive calculations that should only run after user input settles.

Today, developers rely on userland hooks, ad-hoc implementations, or external
utilities like Lodash. These solutions often:

- break under **Strict Mode’s intentional double-invocation**, leaking timers or
  firing twice,
- behave unpredictably with **Concurrent Rendering**, where stale closures or
  premature effects can slip through,
- fail to guarantee proper **cleanup** when dependencies change rapidly,
- diverge in semantics (different interpretations of leading/trailing/maxWait),
  making learning inconsistent.

By providing a **first-class, officially supported hook**, React can:

- ensure correctness aligned with its lifecycle and scheduling model,
- reduce subtle bugs that even experienced developers make,
- simplify teaching: one canonical API instead of dozens of competing recipes,
- and integrate better with future concurrency features.

Even if this RFC is not accepted, the underlying motivation remains: React
developers need clear, safe, and modern guidance for implementing debounce
without relying on fragile workarounds.

# Detailed design

# API surface

```ts
// Debounced value: returns a value that updates after a quiet period
export function useDebounce<T>(
  value: T,
  delay: number,
  options?: {
    leading?: boolean; // default: false — emit immediately at burst start
    trailing?: boolean; // default: true  — emit after quiet period
    maxWait?: number; // default: undefined — force an update at most every N ms
  },
): T;

// Debounced callback: returns a stable function you can pass to listeners
export function useDebouncedCallback<A extends any[]>(
  fn: (...args: A) => void,
  delay: number,
  options?: {
    leading?: boolean; // default: false
    trailing?: boolean; // default: true
    maxWait?: number; // default: undefined
  },
): (...args: A) => void;
```

**Defaults:** { leading: false, trailing: true } and maxWait omitted.

---

# Behavioral semantics

- Burst window. Any sequence of changes/calls closer than delay ms is considered
  one _burst_.
- Leading / trailing.
  - leading: true → fire once at the start of a burst.
  - trailing: true → fire once at the end of a burst (after delay ms of
    inactivity).
  - If both are true, fire at start and at end (at most twice per burst).
- maxWait. During a very long continuous burst, ensure at least one invocation
  every maxWait ms. Resets when an invocation fires.
- Identity stability.
  - useDebouncedCallback returns a function whose identity is stable across
    renders unless delay or options change.
  - useDebounce returns the last committed debounced value; it does not emit
    intermediate values between commits.
- Strict Mode. Timers are created in effects and fully cleaned up on unmount and
  on re-mount; intentional double-invocation must not cause duplicate
  user-visible calls or timer leaks.
- Concurrent Rendering. No side effects occur for renders that do not commit.
  Debounced emissions happen after commit, respecting the chosen
  leading/trailing semantics.
- SSR. No timers are scheduled on the server. All timing begins on the client
  after hydration.
- Cleanup semantics.
  - On unmount, pending timers are canceled by default (no forced trailing
    flush).
  - When delay/options change, pending timers are canceled and rescheduled using
    the new configuration.
- Error propagation. Exceptions thrown by the debounced callback propagate to
  the caller’s environment when the callback finally runs.

---

# Edge-case matrix

| leading | trailing | Effect per burst                                                            |
| :-----: | :------: | --------------------------------------------------------------------------- |
|  false  |   true   | Default. Emit once after quiet period.                                      |
|  true   |  false   | Emit immediately at burst start only.                                       |
|  true   |   true   | Emit immediately, then again at quiet period end.                           |
|   any   |   any    | With maxWait, ensure at least one emit every maxWait ms during long bursts. |

Notes:

- If trailing is false, maxWait only matters when leading is also true.
- Rapidly changing fn or value is supported: the latest refs are used when the
  emit occurs.

---

# Usage examples

Value debouncing (search):

```ts
const [q, setQ] = useState("");
const dq = useDebounce(q, 300);
useEffect(() => {
  if (dq) fetch(`/api?q=${encodeURIComponent(dq)}`);
}, [dq]);
```

Callback debouncing (resize):

```ts
const onResize = useDebouncedCallback(() => recalcLayout(), 120, {
  trailing: true,
  maxWait: 1000,
});
useEffect(() => {
  window.addEventListener("resize", onResize);
  return () => window.removeEventListener("resize", onResize);
}, [onResize]);
```

Leading + trailing:

```ts
const saveDraft = useDebouncedCallback(syncToServer, 500, {
  leading: true,
  trailing: true,
  maxWait: 5000,
});
```

---

# Implementation sketch (non-normative)

- Feature flag: enableUseDebounce (experimental only). Exports live behind
  experimental entry points.
- Refs & timers:
  - Keep refs for lastInvokeTime, timerId, maxWaitTimerId, pendingArgs,
    pendingValue, optionsRef, and fnRef.
  - Update fnRef.current = fn every render; do not recreate the debounced
    wrapper unless delay/options change.
- Scheduling:
  - On call/change: record pendingArgs/pendingValue, compute whether a leading
    invoke should happen (no active burst), otherwise (re)start a delay timer.
  - On timer fire: if in a burst and trailing, invoke with the latest ref; clear
    timers and timestamps.
  - If maxWait is set and no invoke occurred since lastInvokeTime >= maxWait,
    force an invoke.
- Effects & cleanup:
  - Use an effect to install timers and return a cleanup that cancels both delay
    and maxWait timers.
  - On unmount or when delay/options change, cleanup cancels timers.
- Stable callback:
  - Return a memoized function that reads from refs and manipulates timers.

Pseudocode (for useDebouncedCallback):

```ts
function useDebouncedCallback(
  fn,
  delay,
  opts = { leading: false, trailing: true },
) {
  const fnRef = useRef(fn);
  fnRef.current = fn;
  const cfgRef = useRef(normalize(opts, delay));
  useEffect(() => {
    cfgRef.current = normalize(opts, delay);
  }, [delay, opts.leading, opts.trailing, opts.maxWait]);

  const state = useRef({
    timer: null,
    maxTimer: null,
    lastInvoke: undefined,
    pending: null,
  });

  useEffect(
    () => () => {
      clear(state.current.timer);
      clear(state.current.maxTimer);
    },
    [],
  );

  const debounced = useCallback(
    (...args) => {
      const now = Date.now();
      const cfg = cfgRef.current;
      const s = state.current;
      const isInBurst =
        s.lastInvoke !== undefined && now - s.lastInvoke < cfg.delay;
      const shouldLead = cfg.leading && !isInBurst && s.timer == null;

      s.pending = args;

      if (shouldLead) {
        fnRef.current(...s.pending);
        s.lastInvoke = Date.now();
        s.pending = null;
      }

      clear(s.timer);
      if (cfg.trailing) {
        s.timer = setTimeout(() => {
          if (s.pending) {
            fnRef.current(...s.pending);
            s.lastInvoke = Date.now();
            s.pending = null;
          }
          s.timer = null;
        }, cfg.delay);
      }

      if (cfg.maxWait != null && !s.maxTimer) {
        s.maxTimer = setTimeout(() => {
          if (s.pending) {
            fnRef.current(...s.pending);
            s.lastInvoke = Date.now();
            s.pending = null;
          }
          clear(s.timer);
          s.maxTimer = null;
          s.timer = null;
        }, cfg.maxWait);
      }
    },
    [delay, opts.leading, opts.trailing, opts.maxWait],
  );

  return debounced;
}
```

useDebounce(value, ...) can be built on top of useDebouncedCallback by storing
an internal state and setting it via the debounced callback.

---

# Interactions & guarantees

- No accidental double calls in Strict Mode.
- No stale closures: latest fn/value read from refs.
- No SSR timers: safe to import in SSR.
- No breaking changes: entirely additive and experimental.

---

# Testing strategy (non-normative)

- Fake timers for deterministic timing tests.
- Matrix over {leading, trailing} × maxWait presence.
- Strict Mode double-mount tests.
- SSR/hydration: ensure no timers created on server.
- Identity stability: listener re-subscription does not occur unless config
  changes.

# Drawbacks

- **Implementation cost.** Adding new hooks increases the React codebase surface
  area, which means more code to maintain, test, and support across environments
  (web, React Native, experimental runtimes).
- **Userland viability.** Debouncing can already be implemented in user space
  with small, well-tested hooks or libraries. Including it in core may set a
  precedent for adding many other utility hooks (throttle, memoize, etc.).
- **Teaching complexity.** Introducing a new built-in hook expands the API
  surface developers must learn. It could blur the lines between React’s core
  responsibilities (rendering, scheduling, state management) and general UI
  utilities.
- **Integration risks.** We must ensure this feature integrates smoothly with
  current and planned features (e.g., transitions, concurrent rendering, future
  scheduling APIs). A poorly aligned debounce API could conflict with or obscure
  those patterns.
- **Migration cost.** While this is an additive feature (no breaking changes),
  many teams already rely on custom debounce hooks or libraries. Introducing a
  core version may create “two competing standards” and require teams to decide
  whether to migrate.

In summary, the tradeoff is between offering a **first-party, safe, canonical
API** versus keeping React lean and letting the ecosystem continue to provide
debounce utilities.

# Alternatives

- **Status quo (do nothing).**  
  Keep debouncing in userland. Developers can continue to use existing hooks or
  libraries (e.g., lodash, `usehooks-ts`). This avoids increasing React’s
  surface area, but leaves room for inconsistency and subtle bugs in Strict Mode
  or concurrent rendering.

- **Official documentation recipes.**  
  Instead of a new API, React could provide an “official” example of a safe
  debounce hook in the docs. This would give developers a canonical reference
  without introducing a new built-in hook.

- **Bless a community library.**  
  The React team could recommend a well-maintained debounce hook from the
  ecosystem. This reduces duplication but introduces an external dependency and
  less control over long-term stability.

- **Use transitions instead.**  
  Some cases (like deferring visual updates while typing) can be solved with
  `useTransition`. However, transitions do not provide strict debounce semantics
  (time-based suppression of calls), so they are not a full replacement.

- **Throttling or rate limiting APIs.**  
  Alternative designs could add more generic scheduling primitives (throttle,
  rate limit). While powerful, this would broaden React’s API surface
  significantly and risk scope creep.

**Impact of not doing this:**

- Developers will continue writing or importing debounce utilities, with varying
  correctness.
- Subtle bugs in concurrent/Strict Mode may persist due to ad-hoc
  implementations.
- React loses the chance to provide a unified, reliable solution for one of the
  most common UI patterns.

# Adoption strategy

- **Breaking changes:** None. This feature is purely additive. Existing
  applications and libraries will continue to work without modification.

- **Opt-in adoption:** Developers can gradually replace their existing debounce
  utilities with `useDebounce` or `useDebouncedCallback`. Because the API shape
  is similar to common community hooks, migration should be straightforward.

- **Codemods:** A codemod could help migrate simple cases from popular libraries
  (`usehooks-ts`, `ahooks`, lodash debounce wrappers) to the new React API.
  However, migration is optional — teams can keep their existing solutions if
  desired.

- **Coordination with libraries:**
  - Documentation for React should clarify when to prefer the built-in API
    versus transitions or custom utilities.
  - Popular hook libraries could align with the React API surface to minimize
    confusion and encourage consistency.

- **Experimental rollout:**
  - Initially ship behind a feature flag in the experimental channel.
  - Gather feedback from early adopters and the community.
  - Promote to stable only after validating ergonomics, semantics, and ecosystem
    fit.

In short, adoption would be **incremental and low-risk**: developers can try the
new API immediately without breaking existing code, and the ecosystem can align
gradually.

# How we teach this

## Naming and terminology

- **Primary names:** `useDebounce` (value-focused) and `useDebouncedCallback`
  (callback-focused).
  - Rationale: aligns with common ecosystem terms, immediately communicates
    behavior.
- **Key distinctions to teach:**
  - **Debounce** = wait until input settles, then run once.
  - **Throttle** = run at most once per interval.
  - **Transition** = lower-priority rendering; not time-based suppression.

## Positioning within React concepts

- Present as a **focused timing utility** that complements state/effects — not a
  replacement for state management or transitions.
- Emphasize **Strict Mode** and **Concurrent Rendering** compatibility as the
  core value vs. ad-hoc solutions.

## Documentation plan

- **API Reference pages** for both hooks:
  - Clear parameter tables, defaults (`leading: false`, `trailing: true`,
    `maxWait: undefined`).
  - Behavior timeline diagrams for leading/trailing/maxWait.
  - SSR and cleanup semantics (no timers on server, cancel on unmount/change).
- **Guides:**
  - _Debounce vs Throttle vs Transition_ — side-by-side comparison and decision
    checklist.
  - _Common patterns_: search-as-you-type, resize handlers, draft autosave,
    expensive calculations.
  - _Pitfalls & Gotchas_: stale closures, Strict Mode double-invoke, event
    pooling; how these are handled.
- **Examples (copy-paste):**
  - Minimal “search box” example (value debounce).
  - Event listener example (debounced callback for resize/scroll).
  - Leading + trailing + optional `maxWait` example.
  - Accessibility note: ensure debouncing does not hide critical feedback.

## Teaching new developers

- Introduce after they learn `useState`/`useEffect`.
- Use a simple mental model: “group rapid changes into bursts; emit at start
  and/or end of a burst.”
- Provide an interactive playground showing keystrokes vs. debounced emissions
  on a timeline.

## Teaching existing developers

- Migration note: replacing most userland debounce hooks is a one-line swap.
- Checklist to decide usage:
  - Need to **delay** work until input settles? → `useDebounce` /
    `useDebouncedCallback`.
  - Need to **defer priority** (but not suppress frequency)? → `useTransition`.
  - Need to **cap frequency**? → consider throttle (userland).
- Performance & UX guidance:
  - Choose `delay` thoughtfully; prefer small values (100–300ms) for responsive
    UIs.
  - Use `leading` for perceived responsiveness (e.g., draft save), `trailing`
    for correctness (latest value).

## Documentation changes

- Add a new “Timing & Scheduling Patterns” section under “Managing State” or
  “Effects”.
- Cross-link from existing guides that currently show ad-hoc debouncing recipes.
- Add a glossary entry for “burst window” and link it from both APIs.

## Teaching format and artifacts

- Short video/gif showing typing vs. debounced fetches.
- Timeline diagrams for each option combo (default, leading-only,
  leading+trailing, with `maxWait`).
- Sandbox templates for quick experimentation (search form, resize handler,
  autosave).

# Unresolved questions

- **Trailing on cleanup:** When a component unmounts or options change
  mid-burst, should a pending trailing call be **flushed** or **canceled**?
  (Current proposal: cancel.)
- **`maxWait` default & semantics:** Should `maxWait` remain `undefined` by
  default? If provided, should it reset on both leading and trailing invokes, or
  only on successful invokes?
- **Naming:** `useDebounce` + `useDebouncedCallback` vs. `useDebouncedValue`
  (clearer for value form?). Any risk of confusion with community naming?
- **Identity guarantees:** Is the callback identity strictly stable across
  renders unless `delay/options` change, or should we allow advanced opt-outs
  (e.g., `stable: false`)?
- **Event objects:** For React Synthetic Events passed into debounced callbacks,
  what guidance is required (e.g., read fields synchronously or persist/clone)?
  Should the docs enforce best practices?
- **Transitions interplay:** Should examples encourage `startTransition` inside
  the debounced callback for non-urgent rendering, or keep concerns separate?
- **Priority/scheduling alignment:** Do we need hooks into internal priorities
  (present/future) so debounce emissions can opt into specific lanes/queues?
- **SSR/hydration nuances:** Any edge cases where hydration mismatches could
  occur if the value form (`useDebounce`) feeds into layout-sensitive work?
- **React Native & alternative runtimes:** Timer precision and background
  throttling differ on mobile—do we need platform-specific notes or fallbacks?
- **DevTools visibility:** Should DevTools surface debounced emissions
  distinctly (e.g., annotate updates as “debounced”)?
- **Error surfaces:** If the debounced callback throws, do we need additional
  guidance on error boundaries vs. async error propagation?
- **Abort/cancel API:** Do we expose an escape hatch to **flush** or **cancel**
  programmatically (e.g., returning `{ runNow, cancel }`), or is that
  out-of-scope for v1?
- **Performance budget & code size:** What is the acceptable code size increase
  for core/experimental bundles? Any micro-optimizations needed for hot paths?
- **Docs scope:** Do we include throttle guidance alongside debounce, or keep
  throttle as ecosystem territory to limit scope creep?
- **Telemetry/feedback loop:** Do we need optional signals (in experiments) to
  learn common option mixes (`leading/trailing/maxWait`) before stabilizing?
- **Accessibility guidance:** Any recommended patterns to avoid delaying
  critical feedback (e.g., validation) or ARIA live region updates?
- **Alternative API shapes:** Should we consider a single multipurpose hook
  (e.g., `useRateLimit({ mode: 'debounce' | 'throttle' })`) vs. specialized
  debounce-only APIs?
