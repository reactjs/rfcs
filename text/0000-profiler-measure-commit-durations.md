- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

This proposal adds two new callback props to `<Profiler>` components: `onCommit` and `onPostCommit`. These callbacks will receive information about how much time was spent in the most recent commit for layout effects (`useLayoutEffect` as well as `componentDidMount`, `componentDidUpdate`, and `componentWillUnmount`) and passive effects (`useEffect`).

# Basic example

```jsx
<Profiler
  id="Navigation"
  onRender={recordRenderDurations}
  onCommit={recordLayoutEffectDurations}
  onPostCommit={recordEffectDurations}
>
  {/* Components being profiled */}
</Profiler>
```

# Motivation

React's [Profiler API](https://reactjs.org/docs/profiler.html) currently only measures performance of components during the ["render phase"](https://reactjs.org/docs/strict-mode.html#detecting-unexpected-side-effects). Render phase performance is important, but "commit phase" performance is arguably even more important since it can't be time sliced and blocks the browser from painting or executing other JavaScript.


# Detailed design

### What new fields are required?
* Add a `stateNode` field to Profiler Fibers with `effectDuration` and `passiveEffectDuration` attributes.

### How do we measure and track the new durations?
At a high level:
* React will reset effect durations for each Profiler [during the render phase](https://github.com/facebook/react/blob/64aae7b06fa47e126b0ba9c0ba9896caa803528e/packages/react-reconciler/src/ReactFiberBeginWork.js#L573-L590).
  * This provides DevTools a chance to read layout effect durations from its commit hook.
  * A new DevTools hook will need to be added for passive effects though, as those are usually not measured before the commit hook is called (except for unmount cases).
* React will measure time spent in `use*Effect` and `componentDid*` methods by calling `performance.now()` (or `Date.now()` if the Performance API is unavailable) before and after executing user code.
  * Each measurement will be added to the nearest Profiler `stateNode`.
  * To find the nearest Profiler Fiber, React will walk the `return` tree.
* Each Profiler will later propagate the durations up to its next nearest Profiler ancestor [after calling user callbacks](https://github.com/facebook/react/blob/64aae7b06fa47e126b0ba9c0ba9896caa803528e/packages/react-reconciler/src/ReactFiberCommitWork.js#L594-L621).

#### Layout effect durations
Measuring layout effects and calling `onCommit` is fairly straight forward:
* During the commit phase:
  * React will use the Profiler timer to measure the duration of layout effects (and class `componentDid*` lifecycle methods).
  * React will call Profiler `onCommit` props with accumulated durations.
  * React will then bubble durations to the next nearest Profiler.

#### Passive effect durations
Measuring and reporting passive effects is more complex than layout effects. In particular, deciding on *when* to call `onPostCommit` is tricky because:
1. Passive effects won't have been processed by the time we are committing the tree (at least not for the render work we're currently committing). Duration values will represent the previous round of passive effects.
1. We *could* call the previous `onPostCommit` callback at this time, although that presents additional problems since memoized props and interactions may have changed since then:
   1. The `onPostCommit` callback may have changed in the most recent render.
   1. The `memoizedInteractions` set corresponds to the current batch of render work, so it would be incorrect.
   1. The `id` of the Profiler also corresponds to the current batch of render work. (It may have changed since the render our passive effects are part of.)
   1. This would make resetting the passive effect durations a little more complicated as well. I think we should do this during the begin phase, so DevTools has a chance to read these values, but if we did that we'd erase the previous passive effect durations too.

Measuring passive effects and calling `onPostCommit` is more complex then:
* React will use the Profiler timer to measure the duration of passive effects while flushing in preparation for a new render phase.
* Once all passive effects have been flushed, React will bubble any accumulated durations up through ancestor Profilers and call any registered `onPostCommit` callbacks.
  * This must be done before calling `flushSyncCallbackQueue()` in case the flushed effects schedule more work.

In order to allow the DevTools Profiler to record these durations we will also need to add a new DevTools hook. (Normally DevTools reads values in its commit hook, but this won't work for passive effect durations for reasons mentioned above.)

#### Before mutation durations

Currently this category only includes the class component lifecycle `getSnapshotBeforeUpdate`, but (we will probably add a before-mutation hook at some point too).

Given the currently proposed callbacks (`onRender`, `onCommit`, `onPostCommit`) I believe it's best to leave before mutation durations for a future RFC (e.g. `onPreCommit`). Pre-mutation effects aren't very common, so this seems less urgent.

#### Effect cleanup functions
Measuring cleaning functions also presents a few options, (since cleanup functions don't get run until an update or unmount- after we've already reported durations).

There are a couple of options:
1. ☆☆☆ Delay reporting commit durations until the *next* commit (after cleanup functions have been run) so the durations are inclusive of both. This has similar complexity downsides as mentioned above for passive effects though.
1. ★☆☆ Add new callbacks for cleanup durations (e.g. `onCommitCleanup`? `onCommitDestroy`?) and report them separately. This would provide the most insight into where slowness comes from, but I also think this might be overkill.
1. ★★☆ Don't measure these durations. Typically cleanup functions are small (e.g. removing an event listener). This would be a continued blindspot in the Profiler API though.
1. ★★★ Measure cleanup functions, but include their durations as part of the current commit. An argument could be made for this being the *best* way to think of them, since their durations impact the *current* commit (not the previous one), although it might be confusing since e.g. interactions are for the current commit.

For this RFC, I am proposing the 4th option above.

### What info do the new callbacks receive?
The new callbacks will receive the following parameters:
* **`id: string`** - The id prop of the Profiler tree that has just committed. This can be used to identify which part of the tree was committed if you are using multiple profilers.
* **`phase: "mount" | "update"`** - Identifies whether the tree has just been mounted for the first time or re-rendered due to a change in props, state, or hooks.
* **`duration: number`** - Total duration of all user effects (or class lifecycle methods) for the most recent commit phase.
* **`commitTime: number`** - Start time of the current commit (shared between all Profilers in the commit, enabling them to be grouped if desirable).
* **`interactions`**: Set - Set of “interactions” that were being traced the update was scheduled (e.g. when render or setState were called).

# Drawbacks

The main drawback to doing this is that Profiling adds memory and CPU overhead. I think the design of this API will limit the impact of both though.

### Memory
Memory impact should be minimal since it only impacts Profiler Fibers. (There should be no impact at all for the production bundle or for apps that don't use the Profiler API.)

### CPU/time: 
Recording effect durations and finding the nearest Profiler Fiber will add some additional CPU overhead to the commit phase, but it should hopefully be minimal:
* `performance.now()` calls are fast and there should not be many of these in the common case.
* Traversing the tree to find the nearest Profiler fiber should typically be a logarithmic operation.

The above will also only impact the commit phase for portions of the tree that are in Profile mode. Apps that don't use the Profiler API will not incur the overhead.

# Alternatives

* Rather than `onCommit` and `onPostCommit` we could name the new callback props `onLayoutEffects` and `onEffects`.
* An alternative to walking the tree to find the nearest Profiler ancestor could be to store a pointer on each Fiber to the nearest Profiler Fiber. This could save some tree traversal time, but at the expense of adding a new field to every Fiber which would negatively impact memory.

# Adoption strategy

Both new `Profiler` callback props will be optional. Existing callsites will continue to report render phase duration and can be gradually upgraded in place to also report commit phase duration as desired.

# How we teach this

I will write a React block post (or similar) with examples of how to use the new APIs and what they mean.

# Unresolved questions

* Will the proposed implementation be able to handle cascading updates with renders and commits?
* Is it okay to not report time spent in clenaup functions for certain unmount scenarios?