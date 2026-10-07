- Start Date: 2026-10-02
- RFC PR:
- React Issue:

# Summary

Expose a small set of global numeric React work counters for production measurement: updates, component renders, `setState` calls, commits, and ticks. Allow tooling to collect these metrics per commit or per tick.

The counters record events where React already performs the work. The resulting numbers help app teams see whether an interaction causes more work after a release, a React upgrade, or a compiler configuration change.

The purpose is to complement user-experience timings with a direct measure of the amount of React work. This proposal is limited to numeric counters; it does not introduce a tracing or attribution system.

# Basic example

## A controlled example

Consider a list of 100 rows. Each row reads `theme` from a context that also contains an unread notification count. Updating the unread count changes the context value and causes the rows to render, although their displayed theme stays the same.

In our test, splitting the context changes the average component executions per interaction from 102 to 2, with one commit in either version. Both versions produce the same final DOM and can feel responsive on a fast device.

In isolation, this can look like an obvious problem to avoid. Large codebases rarely arrive at it through one deliberate choice. A context gains another field, a shared hook acquires more consumers, and components built by different teams become connected over years of changes. The resulting code can be far from good, even if each change made sense in its original context.

These patterns can stay hidden for years. The UI remains correct, fast devices absorb the extra work, and nobody has a complete view of the dependencies. Recognizing the problem in a small example is much easier than finding it in an application with that history.

An app team could compare these counts between rollout cohorts alongside INP. The render count makes the change in work visible even when the timing difference is small. Detailed profiling can then explain which components account for it.

## A production interaction with many moving parts

Now imagine opening a details panel in a large enterprise app. The interaction touches routing, permissions, shared data stores, several context providers, and components maintained by different teams. Feature flags change which parts of the tree are present. The engineer looking at the dashboard may not know which components should render, or even which updates opening the panel triggers.

A new release reaches a small rollout cohort. The panel looks the same, and timing metrics show no clear change. There are several plausible explanations: the release may have added work, changed when existing work happens, or had no meaningful effect. Reading the diff alone doesn't settle what happened across the assembled application.

Suppose the rollout cohort shows twice as many component executions per panel opening, while state-setter calls and commits remain similar. That doesn't identify a faulty provider or prove the extra renders are unnecessary. It does establish a useful observation: a comparable interaction is executing more components without a corresponding increase in setter calls or commits. The team has a reason to investigate before expanding the rollout, even while timings remain inconclusive.

If setter calls and commits also increase, the investigation starts from a different observation: the interaction is requesting more state changes and committing more often. Either way, the engineer can take a specific question to the teams involved and try to reproduce it with detailed profiling.

This is where basic counters help most. We don't need to know the correct render count in advance, have a complete mental model of the application, or already know the fix. We need to see that the work changed. The cause can still be unknown when the signal becomes useful.

## Observability before investigation

The [React DevTools Profiler](https://react.dev/reference/react/Profiler) already helps us inspect a recording: how many commits occurred, which components rendered, and how much rendering time they took. That is useful once we record the interaction we want to understand.

What we want to observe routinely in production is:

- How many ticks of work preceded each commit, and how much work occurred in each tick?
- How many commits exceeded an application's work budget during initial load, after a user action, or during a transition?
- Did those patterns change after a release, even before anyone noticed a slowdown?

A team can investigate these questions with detailed profiling and additional instrumentation. But that investigation often starts only after someone senses a problem. They then inspect how the app uses `useTransition` or `useDeferredValue`, or follow the updates triggered by callbacks and effects. Until then, changes in the amount of work can remain invisible.

The proposed metrics establish an observability primitive for this earlier stage. Tooling can collect work counts per tick and per commit, mark application-defined load, action, or transition windows, and compare the results against a baseline or work budget. The counters supply the work measurements; the application supplies the context. This makes it possible to detect a change and decide where detailed investigation is needed before the problem becomes noticeable.

# Motivation

## Timings and work counts answer different questions

When we look after a large React app, we rely on production metrics like Core Web Vitals (LCP, INP, CLS), TTVC, and long tasks. These help us understand what people experience. When an interaction gets slower, there's another question we'd like to answer: **did the app ask React to do more work?**

That can be hard to tell from timings alone. The same work takes different amounts of time on different devices. Network conditions, extensions, and CPU throttling add variation. Some changes also increase the work without making an interaction noticeably slower yet:

- **Fast devices have room to spare.** A tree might render three times as many components after a change and still feel responsive on a developer's laptop.
- **Scheduling keeps interactions responsive.** Transitions and yielding let React spread work out so that the browser can respond to input. Responsiveness can remain good even when the total work increases.
- **Small increases can accumulate.** An extra context consumer or an unstable dependency may have little effect on its own. Across many features, those changes can leave less room for a larger dataset or a slower device.

Timings describe the user-visible outcome. Counts describe how many updates, component executions, state-setter calls, and commits occurred. Together, they help us decide where to investigate.

For controlled interactions, counts can be less sensitive to device speed than timings. Scheduling can still affect the amount of work, so counts are not guaranteed to be identical across devices. They nevertheless give us another signal to compare across builds and rollout cohorts.

## Our production use case

We work on React performance at Atlassian, including Jira and related products. Our codebase uses client rendering, SSR, and hydration. Many teams contribute to the same React tree, and changes reach users through feature-flagged progressive rollouts.

We maintain a patch to collect work counts and use them as guardrail metrics for rollouts and React upgrades. We already collect metrics per commit and per tick in Avalanche, our tooling for inspecting React work. This RFC asks for those basic counters and collection boundaries.

These numbers help us distinguish several situations:

- A rollout causes more components to render for the same interaction.
- An effect adds state-setter calls and another commit.
- A React upgrade changes how many updates or renders occur.
- Compiler adoption reduces component executions on a route.
- An interaction test catches an increase in work even when the UI output stays the same and timings are inconclusive.
- A coding agent compares counts before and after an edit to understand its effect on rendering. A passing functional test or an unchanged screenshot can leave that impact invisible; work counts give the agent concrete feedback about what its change caused.

The expected outcome is that app teams can make these comparisons without maintaining a patched React build or walking the committed tree to reconstruct the counts.

## Where the counts are useful

- **A/B tests and rollout guardrails.** Compare component executions per interaction between cohorts and investigate an increase before expanding a rollout.
- **CI budgets.** Assert work counts for controlled interaction tests where timing measurements are noisy.
- **Fleet dashboards.** Compare work per interaction across products, keeping the interaction and React version explicit.
- **React upgrades.** Compare counts between React versions on real traffic to identify changes that a local benchmark may not capture.

The counts are useful together. A state-setter call does not necessarily produce a render, and multiple updates can be batched into one commit. More state-setter calls suggest a different investigation from the same number of calls causing many more components to render.

## Small examples of the distinction

The following measurements cover four rendering patterns. Each before-and-after pair produces the same final DOM for the interaction, with different amounts or patterns of React work.

We measured these with our patched React 19.2.0. In these tests, development and production builds give identical results. Each scenario has 100 rows and runs 10 interactions.

1. **Derived state synced in an effect.** Search results are stored in state and updated from `useEffect` when the query changes. Each keystroke commits once with the previous results, then again with the updated results.
2. **Inline props passed to memoized rows.** The parent passes a new `style` object and callback on each render.
3. **The same props, with a debug ref write.** Scenario 2 includes `renders.current += 1` during render. In this test, the write causes the compiler to skip the component, illustrating a change in compilation coverage.
4. **Non-responsible state management.** A memoized settings context holds both `theme` and `unread`. Each row is wrapped in `memo` and reads only `theme`. Updating the unread count changes the context value, so every row renders.

Each cell below is **component render executions / commits, averaged per interaction**. The render count includes executions throughout the example, including parent components and repeated executions of the same component. It measures how often component render code runs. The commit count measures how often React commits the result.

For example, 664 component render executions and 20 commits across 10 interactions normalize to **66.4 / 2**. All values and comparisons below use this per-interaction basis.

**Baseline** is the original code. **After manual optimization** is the result of investigating and fixing a noticed performance problem.

| Scenario                            | Baseline     | After manual optimization | Baseline + React Compiler | After manual optimization + React Compiler |
| ----------------------------------- | ------------ | ------------------------- | ------------------------- | ------------------------------------------ |
| 1. Derived state in effect          | 66.4 / **2** | 65.4 / 1                  | 66.4 / **2**              | 65.4 / 1                                   |
| 2. Inline props vs `memo`           | 101 / 1      | 2.9 / 1                   | **2.9 / 1**               | 2.9 / 1                                    |
| 3. Inline props + debug ref         | 101 / 1      | 2.9 / 1                   | **101 / 1**               | 2.9 / 1                                    |
| 4. Non-responsible state management | 102 / 1      | 2 / 1                     | 102 / 1                   | 2 / 1                                      |

Reading the results:

- **Derived state in an effect:** manual optimization changes **66.4 / 2** to **65.4 / 1**. Most of the difference is in the commit count. The compiler leaves the baseline at **66.4 / 2**.
- **Inline props:** manual optimization changes **101 / 1** to **2.9 / 1**. The compiler achieves the same reduction without manual optimization.
- **Inline props with the debug ref write:** the compiled baseline stays at **101 / 1** because this component is skipped by the compiler. After manual optimization, the result is **2.9 / 1** with or without the compiler.
- **Non-responsible state management:** manual optimization changes **102 / 1** to **2 / 1**. The compiler alone leaves the baseline at **102 / 1**.

All four examples can feel responsive on a fast device. The counts reveal the impact of manual optimization and where the compiler achieves a reduction without that investigation. They give us a measurable difference to investigate even when timings look similar.

## How this fits with React Compiler

[React Compiler](https://react.dev/learn/react-compiler/introduction) addresses the inline-prop example through automatic memoization. Counts let us measure that improvement rather than assume its effect on production traffic.

Other work remains. A component calling `useContext` subscribes to the context value, so a change to that value still causes the consumer to render. The compiler may reuse calculations or JSX inside the consumer, but it does not turn the subscription into a field-level selector. [Context behavior](https://react.dev/reference/react/useContext).

An effect that synchronizes derived state also requests another update. The compiler does not rewrite that behavior into a render-time calculation; React's guidance is to derive the value during render. [Effect guidance](https://react.dev/learn/you-might-not-need-an-effect).

The debug-ref example is a compilation-coverage caveat. Writes during render violate the ref rules, and compiler diagnostics help identify components it cannot safely optimize. It should not be read as a separate class of work the compiler is expected to eliminate. [Compiler diagnostics](https://react.dev/reference/eslint-plugin-react-hooks).

Work counts therefore serve both purposes: measuring the work that remains and measuring the reductions the compiler delivers.

# Detailed design

## Scope

The scope is deliberately small:

- Global counters for update work and component rendering.
- A global counter for `setState` dispatches.
- A global counter for commits: completion of scheduled work.
- A global counter for render ticks: time-budgeted render slices in concurrent mode, or an uninterrupted flush in synchronous mode.
- Independently enabled per-commit and per-tick metric collection.

Instrumentation is limited to simple counter increments at existing execution points.

The counters are global and cumulative across roots. Tooling can read them at any two points and calculate the work between those points without instrumenting each root. Optional per-tick and per-commit collection records numeric snapshots or deltas. It adds no timers, tree walks, component attribution, traces, or retained application objects. Interaction boundaries, comparisons between releases, dashboards, and alerts remain the responsibility of the consuming tooling.

## Counted events

The proposed event definitions are:

| Count | Event |
| ----- | ----- |
| Update work | React processes work on an existing fiber (`current !== null`), as in the patch's `updateWorkCount`. |
| Component rendering | React enters a component render path, as counted by the patch's `renderWithHooks` and class-component counters. |
| State-setter calls | React enters `dispatchSetState`, before an eager no-op bailout can occur. |
| Commits | React commits the result of scheduled work. |
| Render ticks | A render work-loop invocation: a scheduler-budgeted slice in concurrent mode or an uninterrupted synchronous flush. |

The render-path counters include work in attempts that never commit. They describe entries into the instrumented paths, rather than unique components or DOM changes. A render path can include an internal replay or bailout; its count should retain the meaning of the existing instrumentation rather than imply a different event.

The state-setter count records the dispatch, even if React can skip scheduling an update because the state is unchanged. Invoking a functional updater later is not another state-setter call. This distinction follows the separation between calling a setter and React processing the resulting update. [State-setter behavior](https://react.dev/reference/react/useState).

A batched commit increments the commit count once. It does not increment once per component or DOM mutation. A render attempt that never reaches a commit contributes render executions but no commit.

These are event counts, not a promise that one setter call equals one update, render, tick, or commit. Development-only replays count when they execute the instrumented event; comparisons must identify the build mode.

A tick describes one render slice; a commit describes the completion of scheduled work. Concurrent work can span several ticks before it commits. The counters remain global across those boundaries.

## Per-tick and per-commit collection

Global counters are available independently of optional recordings. Tooling can enable per-tick recording, per-commit recording, or both, and collect snapshots or deltas at those boundaries.

Per-tick metrics show the work as it happens, including work before a commit is available. Per-commit metrics let tooling compare completed units of work. This makes it possible to inspect a change at either level and then aggregate the numbers for an interaction or a rollout cohort. We use both views in Avalanche.

The collection boundary determines which unit of work the counts describe. Disabling a recording stops appending to its array without removing the global counters. Consumers need a way to consume or clear recorded entries so that these arrays do not grow indefinitely.

# Drawbacks

The counter increments have negligible overhead. Optional per-tick and per-commit recordings add snapshot allocation and storage. Their arrays grow while collection is enabled, so the interface must let consumers control collection and consume or clear the records.

Counts can also be misread. One expensive component execution may cost more than many inexpensive ones. A lower render count does not necessarily mean better responsiveness, and useful concurrent work can include abandoned render attempts. Teams should interpret counts alongside timings and use profiling to explain a change.

Aggregate counts do not identify the component or update responsible for a regression. This limitation is intentional: the feature supplies a broad signal, while existing tools provide detailed diagnosis.

# Alternatives

## Continue using timings alone

Timings remain the primary evidence of user experience. They can be enough when a regression is large or easy to reproduce. They do not directly distinguish more React work from the same work taking longer, and small changes can be obscured by device and environmental variation.

## Use `<Profiler>`

[`<Profiler>`](https://react.dev/reference/react/Profiler) provides subtree timings and commit callbacks. Production profiling requires a special profiling build. Its callback does not expose the proposed component-execution or state-setter counts.

For commit counts alone, its callback may be sufficient. The gap is collecting the other counts directly, without requiring the broader timing instrumentation.

## Walk the committed tree through the DevTools hook

Our detector uses `onCommitFiberRoot` to inspect committed work. Deriving counts requires a JavaScript tree walk on each commit. The cost grows with the tree, including parts that may have done little work for that update.

The committed tree also cannot recover every component execution in render attempts that were abandoned. Direct increments record executions when they happen.

## Use React Performance tracks

[React Performance tracks](https://react.dev/reference/dev-tools/react-performance-tracks) provide richer information in browser performance traces and are available in development and profiling builds. They are useful for explaining particular interactions. This proposal addresses lightweight numeric aggregation across production traffic.

## Instrument application components and state setters

Application tooling can wrap selected components or setters. That requires coverage across application and dependency code, and component wrappers can themselves affect rendering. React already passes through the relevant events and can count them directly.

## Continue patching React

This is our current approach. It demonstrates the utility of the numbers for our workflows, but leaves each adopting team responsible for adapting the instrumentation across React releases. A supported counter surface would make that measurement accessible without a local patch.

# Adoption strategy

This is an additive measurement feature. Existing applications retain their behavior and do not need source changes or a codemod.

Teams that want the data enable collection through the agreed interface, choose per-commit or per-tick metrics, and connect the counts to their existing interaction tests or production metrics. They can begin with one route or rollout cohort and compare counts alongside their current timing metrics.

Apps using a local React patch can migrate to the supported counts after checking that the event definitions match. There is no requirement for all React apps to adopt the feature.

# How we teach this

Not every team has an SRE, uses observability tools, or needs to make performance a priority. Many applications work well without collecting React metrics, and their developers should not need to learn these counters to build with React.

Teach this as an optional capability for teams that need to understand rendering at scale, monitor rollouts, or check the impact of their changes. Introduce the counters when those questions arise, rather than adding them to the concepts every React developer is expected to know.

Use the terms *update*, *render*, *state setter*, *commit*, and *tick*. Explain exactly which event increments each number, and keep the distinction between them visible. A tick is a render slice, time-budgeted in concurrent mode and uninterrupted in a synchronous flush; a commit completes scheduled work. Explain how to read the global counters and optionally record metrics at either boundary.

A simple example is an effect that sets derived state: it adds a setter call and can add another render and commit. A context change can instead leave the setter count unchanged while causing many more component executions. The two patterns suggest different places to investigate.

Documentation should present the counters as production observability for app and tooling teams. Beginner guidance about rendering, effects, or memoization does not need to change. A focused reference and an example of comparing interaction counts would be sufficient.

The central teaching point is that counts measure event frequency. They complement timings; they do not score code quality or identify a cause on their own.

# Unresolved questions

- How should React expose the global counters and collected per-commit and per-tick metrics?
- How should tooling independently enable or disable per-commit and per-tick recording, and consume or clear the growing arrays of recorded metrics?
