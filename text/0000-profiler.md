- Start Date: 2018-05-22
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

New React profiling component that collects timing information in order to measure the "cost" of rendering.

This component will also integrate with the [experimental `interaction-tracking` API](https://github.com/facebook/react/pull/13234) so that tracked interactions can be correlated with the render(s) they cause. This enables the calculation of "wall time" (elapsed real time) from when e.g. a user clicks a form button until when the DOM is updated in response. It also enables long-running renders to be more easily attributed and reproduced.

Note that an experimental release of the profiler component is available in version 16.4 as `React.unstable_Profiler`. It does not yet support interactions as that package has not been released.

# Usage example

`Profiler` can be declared anywhere within a React tree to measure the cost of rendering that portion of the tree.

For example, to profile a `Navigation` component and its descendants:
```js
render(
  <App>
    <Profiler id="Navigation" onRender={callback}>
      <Navigation {...props} />
    </Profiler>
    <Main {...props} />
  </App>
);
```

Multiple `Profiler` components can be used to measure different parts of an application:
```js
render(
  <App>
    <Profiler id="Navigation" onRender={callback}>
      <Navigation {...props} />
    </Profiler>
    <Profiler id="Main" onRender={callback}>
      <Main {...props} />
    </Profiler>
    </App>
);
```

`Profiler` components can also be nested to measure different components within the same subtree:
```js
render(
  <App>
    <Profiler id="Panel" onRender={callback}>
      <Panel {...props}>
        <Profiler id="Content" onRender={callback}>
          <Content {...props} />
        </Profiler>
        <Profiler id="PreviewPane" onRender={callback}>
          <PreviewPane {...props} />
        </Profiler>
      </Panel>
    </Profiler>
  </App>
);
```

Although `Profiler` is a light-weight component, it should be used only when necessary; each use adds CPU and memory overhead to an application.

# Motivation

It is important that render timing metrics work properly with React's experimental async rendering mode. When asynchronously rendering, React may yield periodically so that an app remains responsive even on low-powered devices. This yielded time (when React is not running) should not be included when considering the "cost" of a render. This distinction is not possible to implement in user space.

Timing measurements should also be significantly lighter weight than the current User Timing API so that they can be gathered in production without negatively impacting user experience. (The User Timing API is currently disabled for production because it is slow.) In addition to a faster implementation, we can further limit the impact on existing apps by creating a new production + profiling bundle. This way, apps that don't make use of the `Profiler` component (or wish to disable it globally) will not incur any additional overhead. (The `Profiler` component will render its children in production mode but its `onRender` callback will not be called.)

# Detailed design

The `onRender` callback is called each time a component within the `Profiler` renders. It receives the following parameters:
```js
function onRenderCallback(
  id: string,
  phase: "mount" | "update",
  actualDuration: number,
  baseDuration: number,
  startTime: number,
  commitTime: number,
  interactions: Array<{ name: string, timestamp: number }>,
): void {
  // Aggregate or log render timings...
}

```

#### `id: string`
The `id` value of the `Profiler` tag that was measured. This value can change between renders if e.g. it is derived from `state` or `props`.

#### `phase: "mount" | "update"`
Identifies whether this component has just been mounted or re-rendered due to a change in `state` or `props`.

#### `actualDuration: number`
Time spent rendering the `Profiler` and its descendants for the current (most recent recent) update. This time tells us how well the subtree makes use of `shouldComponentUpdate` for memoization.

Ideally, this time should decrease significantly after the initial mount as many of the descendants will only need to re-render if their specific `props` change.

Note that in async mode, under certain conditions, React might render the same component more than once as part of a single commit. (In this event, the "actual" time for an update might be larger than the initial time.)

#### `baseDuration: number`
Duration of the most recent `render` time for each individual component within the `Profiler` tree. In other words, this value will only change when a component is re-rendered. It reflects a worst-case cost of rendering (e.g. the initial mount or no `shouldComponentUpdate` memoization).

#### `startTime: number`
Start time identifies when a particular commit started rendering. Although insufficient to determine the cause of the render, it can at least be used to rule out certain interactions (e.g. mouse click, Flux action). This may be helpful if you are also collecting other types of interactions and trying to correlate them with renders.

Start time isn't just the commit time less the "actual" time, because in async rendering mode React may yield during a render. This "yielded time" (when React was not doing work) is not included in either the "actual" or "base" time measurements.

#### `commitTime: number`
Commit time could be roughly determined using e.g. `performance.now()` within the `onRender` callback, but multiple `Profiler` components would end up with slightly different times for a single commit. Instead, an explicit time is provided (shared between all `Profiler`s in the commit) enabling them to be grouped if desirable.

#### `interactions: Array<{ name: string, timestamp: number }>`
An array of interactions that were being tracked (via the `interaction-tracking` package) when this commit was initially scheduled (e.g. when `render` or `setState` were called).

In the event of a cascading render (e.g. an update scheduled from `componentDidMount` or `componentDidUpdate`) React will forward these interactions along to the subsequent `onRender` calls.

# Drawbacks

Overuse of this component might negatively impact application performance.

# Alternatives

None considered.

# Adoption strategy

This is an entirely new component. Adoption can be organic and gradual.

# How we teach this

A reactjs.org blog post would be a good initial start.

Perhaps we could provide some sort of discoverability within React DevTools.

# Related proposals

* [facebook/react/pull/13253](https://github.com/facebook/react/pull/13253): Integration with the proposed `interaction-tracking` package
* [facebook/react-devtools/pull/1069](https://github.com/facebook/react-devtools/pull/1069): Integration with React DevTools