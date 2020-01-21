- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

This proposal adds two new callback props to `<Profiler>` components: `onCommit` and `onPostCommit`. These callbacks will receive information about how much time was spent in the most recent commit for layout effects (`useLayoutEffect` as well as `componentDidMount`, `componentDidUpdate`, and `componentWillUnmount`) and passive effects (`useEffect`).

# Basic example

```js
<Profiler
  id="Navigation"
  onRender={recordRenderDurations}
  onCommit={recordLayoutEffectDurations}
  onPostCommit={recordEffectDurations}
>
  <!-- Components being profiled -->
</Profiler>
```

# Motivation

React's [Profiler API](https://reactjs.org/docs/profiler.html) currently only measures performance of components during the ["render phase"](https://reactjs.org/docs/strict-mode.html#detecting-unexpected-side-effects). Render phase performance is important, but "commit phase" performance is arguably even more important since it can't be time sliced and blocks the browser from painting or executing other JavaScript.


# Detailed design

#### What new fields are required?
* Add a `stateNode` field to Profiler Fibers with `effectDuration` and `passiveEffectDuration` attributes.

#### How do we measure and track the new durations?
* React will reset effect durations for each Profiler [during the render phase](https://github.com/facebook/react/blob/64aae7b06fa47e126b0ba9c0ba9896caa803528e/packages/react-reconciler/src/ReactFiberBeginWork.js#L573-L590).
  * (This assumes we always start at the root and traverse the full tree. Alternately, we could reset each durations immediately after calling `onCommit` or `onPostCommit`.)
* During the commit phase, React will measure time spent in `use*Effect` and `componentDid*` methods by calling `performance.now()` (or `Date.now()` if the Performance API is unavailable) before and after executing user code.
  * Each measurement will be added to the nearest Profiler `stateNode`.
  * To find the nearest Profiler Fiber, React will walk the `return` tree.
* Each Profiler will also propagate the durations up to its next nearest Profiler ancestor [after calling user callbacks](https://github.com/facebook/react/blob/64aae7b06fa47e126b0ba9c0ba9896caa803528e/packages/react-reconciler/src/ReactFiberCommitWork.js#L594-L621).

#### What info do the new callbacks receive?
The new callbacks will receive the following parameters:
* **`id: string`** - The id prop of the Profiler tree that has just committed. This can be used to identify which part of the tree was committed if you are using multiple profilers.
* **`phase: "mount" | "update"`** - Identifies whether the tree has just been mounted for the first time or re-rendered due to a change in props, state, or hooks.
* **`duration: number`** - Total duration of all user effects (or class lifecycle methods) for the most recent commit phase.
* **`interactions`**: Set - Set of “interactions” that were being traced the update was scheduled (e.g. when render or setState were called).

# Drawbacks

The main drawback to doing this is that Profiling adds memory and CPU overhead. I think the design of this API will limit the impact of both though.

#### Memory
Memory impact should be minimal since it only impacts Profiler Fibers. (There should be no impact at all for the production bundle or for apps that don't use the Profiler API.)

#### CPU/time: 
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