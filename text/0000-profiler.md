- Start Date: 2018-05-22
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

New React profiling component that collects timing information in order to measure the "cost" of rendering.

(Note that an experimental release of this component is currently slated for 16.4 as `React.unstable_Profiler`.)

# Basic example

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

It is important that render timing metrics work properly with React's experimental async rendering mode. When asynchronously rendering, React may yield periodically so that an app remains responsive even on low-powered devices. This yielded time (when React is not running) should not be included when considering the "cost" of a render. This distinction is not possible to implement this in user space.

Timing measurements should also be significantly lighter weight than the current User Timing API so that they can be gathered in production without negatively impacting user experience. (The User Timing API is currently disabled for production because it is slow.) In addition to a faster implementation, we can further limit the impact on existing apps by creating a new production + profiling bundle. This way, apps that don't make use of the `Profiler` component (or wish to disable it globally) will not incur any additional overhead.

# Detailed design

The `onRender` callback is called each time a component within the `Profiler` renders. It receives the following parameters:
```js
function onRenderCallback(
  id: string,
  phaseL "mount" | "update",
  actualTime: number,
  baseTime: number,
  startTime: number,
  commitTime: number
): void {
  // Aggregate or log render timings...
}

```

#### `id: string`
The `id` value of the `Profiler` tag that was measured. This value can change between renders if e.g. it is derived from `state` or `props`.

#### `phase: "mount" | "update"`
Either the string "mount" or "update" (depending on whether this root was newly mounted or has just been updated).

#### `actualTime: number`
Time spent rendering the `Profiler` and its descendants for the current (most recent recent) update. This time tells us how well the subtree makes use of `shouldComponentUpdate` for memoization. Ideally, this time should decrease significantly after the initial mount.

#### `baseTime: number`
Duration of the most recent `render` time for each individual component within the `Profiler` tree. This reflects a worst-case cost of rendering (e.g. the initial mount or no `shouldComponentUpdate` memoization).

#### `startTime: number`
Start time identifies when a particular commit started rendering. Although insufficient to determine the cause of the render, it can at least be used to rule out certain interactions (e.g. mouse click, Flux action). This may be helpful if you are also collecting other types of interactions and trying to correlate them with renders.

Start time isn't just the commit time less the "actual" time, because in async rendering mode React may yield during a render. This "yielded time" (when React was not doing work) is not included in either the "actual" or "base" time measurements.

#### `commitTime: number`
Commit time could be roughly determined using e.g. `performance.now()` within the `onRender` callback, but multiple `Profiler` components would end up with slightly different times for a single commit. Instead, an explicit time is provided (shared between all `Profiler`s in the commit) enabling them to be grouped if desirable.

# Drawbacks

Overuse of this component might negatively impact application performance.

# Alternatives

None considered.

# Adoption strategy

This is an entirely new component. Adoption can be organic and gradual.

# How we teach this

A reactjs.org blog post would be a good initial start.

# Unresolved questions

How will people use this? The current proposal is kind of low level and would probably benefit from some reusable abstractions being built on top of it that e.g. aggregate/batch render timings.

How will this feature integrate with React DevTools? I have some ideas but nothing concrete yet to share.