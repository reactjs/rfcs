- Start Date: (2023-01-04)
- RFC PR:
- React Issue:

# Summary

We want to create a new React version of the [Interaction Tracing API](https://gist.github.com/bvaughn/8de925562903afd2e7a12554adcdda16). This will allow developers to use the Profiler API and DevTools Profiler/Timeline to:

1. Watch for performance regressions for specific transitions
2. Understand why a React transition is slow so they can make performance improvements to better user experience

# Basic example

A transition starts when `startTransition` is called and completes when the updated state is committed and painted on the screen, possibly passing through multiple intermediate states in the process.

Add a config object to `startTransition` with the name of the transition to initiate the transition.

```js
function App() {
  const user = getViewingUser();
  const [pageName, setPageName] = useState("homefeed");
  const onNavigate = (pageName) => {
    startTransition(() => setPageName(pageName), { name: pageName });
  };
  return (
    <>
      <NavBar onNavigate={onNavigate} />
      <Page name={pageName} id={user.id} />
    </>
  );
}
```

Add transition callbacks to the root that will be called asynchronously when a transition starts or completes.

```js
const onTransitionStart = (transitionName, startTime) => ...;
const onTransitionComplete = (transitionName, startTime, endTime) => ...;
const onTransitionProgress = (transitionName, startTime, currentTime, pendingSuspenseBoundaries) => ...;
const root = React.createRoot(container, {
  transitionCallbacks: {
    onTransitionStart,
    onTransitionComplete,
    onTransitionProgress,
  },
});
```

Add tracing markers to get more find grained details about when a portion of the page starts rendering and when it finishes rendering.

```js
function Profile({ id }) {
  return (
    <TracingMarker name="profile">
      <Suspense fallback={<LoadingSpinner />}>
        <ProfileHeader id={id} />
        <TracingMarker name="profile:photo-feed">
          <div>Photos</div>
          <Suspense fallback={<LoadingFeed />}>
            <PhotoFeed />
          </Suspense>
        </TracingMarker>
        <TracingMarker name="profile:profile-feed">
          <div>Profile Feed</div>
          <Suspense fallback={<LoadingFeed />}>
            <ProfileFeed />
          </Suspense>
        </TracingMarker>
      </Suspense>
    </TracingMarker>
  );
}
```

Similar to the transition callbacks, add tracing marker callbacks to the root that will be called asynchronously when a tracing marker commits and paints to the screen (possibly passing through multiple intermediate states in the process).

```js
const onMarkerProgress = (transitionName, markerName, startTime, currentTime, pendingSuspenseBoundaries) => ...;
const onMarkerComplete = (transitionName, markerName, startTime, endTime) => ...;
const root = React.createRoot(container, {
  transitionCallbacks: {
    onTransitionStart,
    onTransitionComplete,
  },
});
```

# Motivation

Currently, React has two profiling tools. The old Profiler shows an overview of all the commits in a profiling session. For each commit, it also shows all components that rendered and the amount of time it took for them to render. The new Scheduling Profiler shows when components schedule updates and when React works on these updates. Both of these profiler help developers identify performance problems in their code.

However, we realized that developers don’t find knowing about individual slow commits or components out of context that useful. They want to know about what actually causes the slow commits. They also want to be able to track specific interactions (ex. a button click, an initial load, or a page navigation) to watch for performance regressions and to understand why an interaction was slow and how to fix it.

There is currently no way to answer these questions in open source, which means that OSS developers aren’t able to use the React profiling tools effectively. We previously tried to solve this issue by creating an [Interaction Tracing API](https://gist.github.com/bvaughn/8de925562903afd2e7a12554adcdda16), but it had some fundamental design flaws that prevented it from being as useful as we hoped. In the old interaction tracing API, interactions were tracked at the root. Because of this, when React batches updates together, their interactions would get entangled. Cascading updates would carry these entangled interactions along even if they had completed, resulting in never ending interactions. We ended up [removing this API](https://github.com/facebook/react/pull/20037) because of these issues.

We want to create a new React version of the Interaction Tracing API, the Transition Tracing API. This will allow developers to use the Profiler API and DevTools Profiler/Timeline to:

1. Watch for performance regressions for specific transitions
2. Understand why a React transition is slow so they can make performance improvements to better user experience

# Detailed design

This observation allows us to propose the following API:

A transition **starts from an event** (ex. a user action that schedules an update) **and ends when the updated state is commited** (ex. showing some content on a page), possibly passing through multiple intermediate commits. You can navigate **from** an event to multiple **markers** (ex. clicking on the Facebook homepage leads to Stories and Newsfeed being loaded). Because updates with the same lanes can be **batched** together, an update can also start from multiple places (ex. you can click Home, then click Marketplace, these two updates will be batched together, and you would show Marketplace in the end):

This observation allows us to propose the following API:

1.  **Add an optional config object with a `name` field to `startTransition` to initiate the transition.** `startTransition` is what we recommend that developers use to wrap long updates that result from high priority events. Since transitions take a long time to complete, we recommend that they are wrapped with `startTransition` regardless, making this a great place to initiate transitions.
2.  **Add `<TracingMarker name="..."/>` component to define a subtree to be tracked.** We decouple this from Suspense because Suspense boundaries are fragile to refactoring. The tracing marker can be used in two ways:
    - To tell when you’ve reached a point in the tree (ex. you’ve seen the top Nav)
    - To tell when a subtree is loaded (ex. Marketplace is finished loading)
3.  **Add optional `name` field to `Suspense` to get more detailed data about when Suspense boundaries resolve.** This does not define the scope of a transition. It just provides extra data can be used by the transition callbacks to ignore boundaries or to inform tooling like DevTools.
4.  **An interaction finishes when all child suspense boundaries in a tracing marker resolves.** We will store the `interactionIDs` on the nearest suspense boundary below the `<TracingMarker />`. As suspense boundaries resolve, we will migrate the interactions to lower suspense boundaries. Once all suspense boundaries resolve, we will consider the interaction complete and call the transitionCallback with a Complete status.
5.  **Add transition callbacks to the root.**

    - **`onTransitionStart(transitionName: string, startTime: number)`**: We call `onTransitionStart` when a transition is first initiated. `startTime` will be the event time of the event
    - **`onMarkerProgress(transitionName: string, marker: string, startTime: number, currentTime: number pending: Array<{name: null | string}>)`:** We call `onMarkerProgress` when:

      - a `TracingMarker`’s child suspense boundary first commits in a fallback state.
      - a `TracingMarker`’s child suspense boundary resolves.

      The pending array contains the names of the Suspense boundaries that are not yet resolved when the callback was called. It’s an array of objects to support future compatibility for metatdata like bounding rect.

    - **`onTransitionProgress(transitionName: string, startTime: number, currentTime: number, pending: Array<{name: null | string}>)`:** Similar to `onMarkerProgress` except we track the transition starting from the root.
    - **`onMarkerIncomplete(transitionName:: string, marker: string, startTime: number, deletions: Array<{type: string, name?: string, endTime: number}>)`:**
      We call `onMarkerIncomplete` after all `onMarkerProgress` callbacks are called when a tracing marker’s transition is incomplete, ie something unexpected happened during the transition. There are several events that can cause a transition to be incomplete, and multiple of these might happen in a single commit to trigger this function:

      - **The `TracingMarker` (or any child tracing marker) is deleted**. We add `{type: 'marker', name: markerName, endTime: time}` to the `deletions` array
      - **The `TracingMarker`’s name changed.** We add `{type: 'marker', name: markerName, newName: newMarkerName, endTime: time}` to the `deletions` array
      - **A child Suspense boundary that was added during the transition is deleted.** We add `{type: 'suspense', name: boundaryName | null, endTime: time}` to the `deletions` array
      - **An error is thrown.** We add `{type: 'error', boundary: string, error: Error, componentStack: string, endTime: time}` to the `deletions` array.
      - **Other.** Sometimes, as in the case of SSR, we might not know exactly what caused an abort. In this case, we add `{type: 'unknown', endTime: time}` to the deletions array.

      Once `onTransitionIncomplete` is called, the transition can no longer complete (ie. `onTransitionComplete`/`onMarkerComplete` can no longer be called). However, subsequent `onTransitionProgress` callbacks will still be triggered. `onMarkerIncomplete` calls will be propagated up the tree to any parent markers as well as `onTransitionIncomplete`

    - **`onTransitionIncomplete(transitionName: string, startTime: number, currentTime: number, deletions: Array<{type: string, name?: string, endTime: number})`:**
      Similar to `onMarkerIncomplete` except we track the transition from the root.
    - **`onMarkerComplete(transitionName: string, marker: string, startTime: number, endTime: number)`:**
      We call `onMarkerComplete` if all suspense boundaries inside the TracingMarker resolves (or if there are no suspense boundaries), ie if the transition completes as expected. The `endTime` will be the paint time.
    - **`onTransitionComplete(transitionName: string, startTime: number, endTime: number)`:** Similar to `onMarkerComplete` except we track the transition from the root.

# Usage Examples

Here is how we envision someone might build interaction tracing on top of our Transition Tracing APIs

Here’s a small App with a few components, a Profile component and a Homefeed component. We’ll use this app in the examples below.

```js
const container = document.createElement("div");
const root = React.createRoot(container);

function App() {
  const user = getViewingUser();
  const [pageName, setPageName] = useState("homefeed");
  const onNavigate = (pageName) => {
    startTransition(() => setPageName(pageName));
  };
  return (
    <>
      <NavBar onNavigate={onNavigate} />
      <Page name={pageName} id={user.id} />
    </>
  );
}

function NavBar({ onNavigate }) {
  return (
    <>
      <button onClick={() => onNavigate("homefeed")}>Homefeed</button>
      <button onClick={() => onNavigate("profile")}>Profile</button>
    </>
  );
}

function Page({ pageName }) {
  switch (pageName) {
    case "profile":
      return <Profile />;
    case "homefeed":
      return <Homefeed />;
  }
}

function Profile({ id }) {
  return (
    <Suspense fallback={<LoadingSpinner />}>
      <ProfileHeader id={id} />
      <div>Photos</div>
      <Suspense fallback={<LoadingFeed />}>
        <PhotoFeed />
      </Suspense>
      <Suspense fallback={<LoadingFeed />}>
        <ProfileFeed />
      </Suspense>
    </Suspense>
  );
}

function Homefeed({ id }) {
  // ...
}
```

## Trace an interaction

Here’s how you might trace an interaction where you navigate between the Homefeed to the Profile page.

**Step 1: Give the transition a name.** Add a transition name to `startTransition` to let React know that you want to trace a transition. Here we add the name of the page to `startTransition`.

```js
function App() {
  const user = getViewingUser();
  const [pageName, setPageName] = useState("homefeed");
  const onNavigate = (pageName) => {
    startTransition(() => setPageName(pageName), { name: pageName });
  };
  return (
    <>
      <NavBar onNavigate={onNavigate} />
      <Page name={pageName} id={user.id} />
    </>
  );
}
```

**Step 2: Mark the area of the page you want to track.** We add a Tracing Marker around the whole Profile page. This lets us track when the entire Profile page is complete (ie. there are no suspense boundaries in the fallback state). In this example, we also use multiple markers around the Photo Feed and the Profile Feed so we can measure how long these parts of the page take to complete, but this isn’t necessary.

Different apps will have different naming conventions for their tracing markers. In this example, we use parentMarker:childMarker syntax to link the markers back to each other during post processing.
n this example, we have multiple markers", "In this example, we use the parentMarker:childMarker syntax" etc – ("but you don't have to")

```js
function Profile({ id }) {
  return (
    <TracingMarker name="profile">
      <Suspense fallback={<LoadingSpinner />}>
        <ProfileHeader id={id} />
        <TracingMarker name="profile:photo-feed">
          <div>Photos</div>
          <Suspense fallback={<LoadingFeed />}>
            <PhotoFeed />
          </Suspense>
        </TracingMarker>
        <TracingMarker name="profile:profile-feed">
          <div>Profile Feed</div>
          <Suspense fallback={<LoadingFeed />}>
            <ProfileFeed />
          </Suspense>
        </TracingMarker>
      </Suspense>
    </TracingMarker>
  );
}
```

**Step 3: Add callback functions to capture transition log events.** There are multiple transition callback functions, but for the basic case we only need two, onMarkerIncomplete and onMarkerComplete:

- onMarkerComplete is called after all fallback suspense boundaries created as the result of a transition resolve (within a tracing marker).
- onMarkerIncomplete is called if something unexpected happens, (ex. a fallback boundary within a transition is added and removed, an error occurs, the tracing marker gets removed).

Different apps will want to process these cases in different ways. here’s an example of how our app will process these cases:

```js
// Called when the transition starts (ie. when the nav bar is clicked)
function onTransitionStart(name, startTime) {
   // we want to save this because if there is a start transition
   // but no end transition, we know the transition has been cancelled
   logInteraction({name, startTime, status: 'start'})
}

// Called when the transition finishes and is incomplete
function onMarkerIncomplete(
  name,
  marker,
  startTime,
  endTime,
  deletions
) {
  let status = 'complete';
  for (let deletion of deletions) {
    // If a marker was deleted, we consider the transition canceled
    if (deletions.type === 'marker' && deletions.name === marker) {
       status = 'cancel';
       break;
    // If an error occured, we mark the status as error
    } else if (deletions.type === 'error') {
       status = 'error';
    }
  }

  // Otherwise, if a suspense boundary is removed, we consider
  // this completed. If we wanted more granular transition information
  // like if a suspense boundary was unexpectedly removed, we can
  // log this separately.

  logInteraction({
    name,
    marker,
    startTime,
    endTime,
    status,
  });
}

// Called when the transition finishes and is complete
function onMarkerComplete(name, marker, startTime, endTime) {
    // Not all transitions with a complete status are complete transitions.
    // For example, if you click from marketplace to profile and then to homefeed
    // really quickly without a rerender, React won't know that those are
    // two separate transitions. However, the user can detect this by
    // checking that the marker matches the transition OR that multiple
    // transitions have the same end time but different start times
   const isCanceled = !marker.includes(name);
   logInteraction({
     name,
     marker: isCanceled ? name : marker,
     startTime,
     endTime,
     status: isCanceled ? 'cancel' : 'complete',
  });
}

...
const root = React.createRoot(container, {
  transitionCallbacks: {
    onTransitionStart,
    onMarkerIncomplete,
    onMarkerComplete,
  }
});
```

For this App, let’s break down potential things that might happen if we click on the Profile button to navigate to Profile:

- **Neither Photo feed nor Profile feed suspends.**

  1. `onMarkerComplete` will be called with a status of complete for all three markers at the same time.

- **Photo feed does not suspend. Profile feed does but eventually resolves:**

  1. `profile:photo-feed` updates and finishes normally → we call `onMarkerComplete` with a status of complete for the `profile:photo-feed` only.
  2. Eventually, `profile:profile-feed` unsuspends and renders normally → we call `onMarkerComplete` with a status of complete for `profile:profile-feed`. We also call `onMarkerComplete` with a status of complete for `profile` because all children have unsuspended.

- **Profile feed errors while rendering. Photo feed suspends in the meantime.**

  1. `profile:profile-feed` errors while rendering → we call `onMarkerIncomplete` and infer a status of error because there is a deletion with type error in the deletions array.
  2. `profile:photo-feed` unsuspends and finishes normally → we call `onMarkerComplete` with a status of complete for the `profile:photo-feed` only. At the same time, we call `onMarkerIncomplete` with a status of error for profile because all children have unsuspended but a child has errored.

- **Click on Profile. Page gets resized and child suspense boundary of Profile Feed removed before everything completes.**

  1. `profile:profile-feed`‘s child suspense boundary gets removed → We call `onMarkerIncomplete` and infer a status of complete based on the deletions array because we only special case incomplete transitions for marker deletions and errors.
  2. `profile:photo-feed` unsuspends and finishes normally → we call ``onMarkerComplete` with a status of complete for the `profile:photo-feed` only. At the same time, we call `onMarkerIncomplete` with a status of complete for `profile` because we only special case incomplete transitions for marker deletions and errors.

- **Photo feed tracing marker gets removed or its name gets changed**

  1. `profile:profile-feed` completes as normal.
  2. `profile:photo-feed` calls `onMarkerIncomplete` and logs the transition with a status of canceled.
  3. profile calls `onMarkerIncomplete`, but because the `profile` isn’t in the deletions array, we log the transition a status of complete

- **We navigate to Homefeed before Profile is complete.**

  1. All tracing markers with still suspended boundaries will call `onMarkerIncomplete` with a status of canceled because the markers will be deleted before completing

- **We navigate to Homefeed too quickly and don’t start rendering the Profile.**

  1. All tracing markers on the Homefeed will call `onMarkerComplete` but with the wrong transition name (ex. marker homefeed with transition profile) and will be marked as canceled

- **We navigate to a page without tracing markers too quickly and don’t start rendering the Profile.**

  1. In the post processing step, we will check to make sure all transitions with status: start have a corresponding complete/incomplete transition status. Those who don’t will be marked as incomplete.

## Complex cases

Here, we have a Profile Photo Modal that is shown when we click a button. We want to trace an interaction where you click on the Profile Photo Modal button and it launches the Profile Photo Modal. However, even though the profile photo modal' is a child of the profile tracing marker, we don’t want the profile photos modal to influence the profile interaction.

```js
function Profile({ id }) {
  return (
    <Suspense fallback={<LoadingSpinner />}>
      <ProfileHeader id={id} />
      <button onClick={() => {setShowProfilePhotoModal(!showProfilePhotoModal)}}>
      {showProfilePhotoModal ?
        ReactDOM.createPortal(
            <Suspense fallback={<LoadingModal />}>
                <ProfilePhotoModal />
            </Suspense>,
            portalContainer
        : null}
      <div>Photos</div>
      <Suspense fallback={<LoadingFeed />}>
        <PhotoFeed />
      </Suspense>
      <Suspense fallback={<LoadingFeed />}>
        <ProfileFeed />
      </Suspense>
    </Suspense>
  );
}

function ProfilePhotoModal() {
   // ...
}
```

**Step 1: Give the transition a name and Mark the area of the page you want to track.** We give the suspense boundary the same name as the transition name, `profile-photo-modal`, to tie the transitions together so we can process them later.

```js
function Profile({ id }) {
  const [showProfilePhotoCloseup, setShowProfilePhotoCloseup] = useState(false);
  return (
    <Suspense fallback={<LoadingSpinner />}>
      <ProfileHeader id={id} />
      <button onClick={() => {
        startTransition(() => {
            setShowProfilePhotoModal(!showProfilePhotoModal)
        }, name: {'profile-photo-modal'});
      }>
      {showProfilePhotoModal ?
        ReactDOM.createPortal(
            <TracingMarker name="profile-photo-modal"}>
                <Suspense name={"profile-photo-modal"} fallback={<LoadingModal />}>
                    <ProfilePhotoModal />
                </Suspense>
            </TracingMarker>,
            portalContainer
        : null}
      <div>Photos</div>
      <Suspense fallback={<LoadingFeed />}>
        <PhotoFeed />
      </Suspense>
      <Suspense fallback={<LoadingFeed />}>
        <ProfileFeed />
      </Suspense>
    </Suspense>
  );
}
```

**Step 2: Add callback functions to capture transition log events**. Because we want access to more fine grained information about the transition status, we use `onMarkerProgress` instead of `onMarkerComplete`. `onMarkerProgress` is called every time a suspense boundary unsuspends. Using this, we can check, every time a suspense boundary unsuspends, whether the only suspense boundaries that are still pending are boundaries in the current transition or not and mark the interaction as complete when all boundaries in the transition are complete.

```js
const completedTransitions = new Set([]);

function onTransitionStart(name, startTime) {
  logInteraction({ name, startTime, status: "start" });
}

// Called every time a suspense boundary unsuspends
function onMarkerProgress(
  name,
  marker,
  startTime,
  marker,
  currentTime,
  pending
) {
  // Suspense boundary names are either null or a string. Pending contains
  // the suspense boundaries still in a fallback state.
  const transitionFinished = !pending.includes(
    (boundaryName) => boundaryName === null || boundaryName === name
  );

  // If the only suspense boundaries that are pending are boundaries with
  // different names than the transition, the transition is considered finished
  // We can replace onMarkerComplete with this function. We add this transition
  // to the completedTransition set.
  if (transitionFinished) {
    completedTransitions.add(name);
    const isCanceled = !marker.split(":").includes(name);
    logInteraction({
      name,
      marker,
      startTime,
      endTime,
      status: isCanceled ? "cancel" : "complete",
    });
  }
}

// Called when the transition finishes and is incomplete
function onMarkerIncomplete(name, marker, startTime, endTime, deletions) {
  // The transition might be incomplete because of something that happened in
  // an ignored subtree. In this case, we ignore the incomplete signal.
  if (completedTransitions.has(name) || !marker.split(":").includes(name)) {
    return;
  }

  let status = "complete";
  for (let deletion of deletions) {
    if (deletions.type === "marker") {
      status = "cancel";
      break;
    } else if (deletions.type === "error") {
      status = "error";
    }
  }

  logInteraction({
    name,
    marker,
    startTime,
    endTime,
    status,
  });
}

const root = React.createRoot(container, {
  transitionCallbacks: {
    onMarkerComplete,
    onMarkerIncomplete,
    onMarkerProgress,
  },
});
```

For this App, let’s break down potential things that might happen if we click on the Profile button to navigate to Profile:

- **Profile completes and the modal never renders.**

  1. `onMarkerProgress` with a status of complete will be called for all three tracing markers in the profile feed. The `profile-photo-modal` tracing marker is never rendered.

- **Profile completes. Then user clicks on the modal.**

  1. `onMarkerProgress` with a status of complete will be called for all three tracing markers in the profile feed. The `profile-photo-modal` tracing marker isn’t rendered until after the Profile completes.
  2. `onMarkerProgress` with a status of complete will be called for `profile-photo-modal`.

- **Profile starts rendering. Profile photo feed completes. The modal is then clicked and starts rendering. Profile feed completes first.**

  1. `profile-photo` completes. `onMarkerProgress` is called with a status of complete for `profile-photo`. `onMarkerProgress` is also called on `profile` and `profile-feed` but both have pending unnamed suspense boundary, so they don’t complete.
  2. Modal is clicked and starts rendering.
  3. `profile-feed` completes. `onMarkerProgress` is called with a status of complete for `profile-feed`. `onMarkerProgress` is also called on `profile`. Because the only pending suspense boundary inside `profile` is `profile-photo-modal`, which is a different interaction, `profile` completes. `onMarkerProgress` is also called on `profile-photo-modal`, but since its pending suspense boundary has the same name as `profile-photo-modal`, it doesn’t complete
  4. `profile-photo-modal` completes. `onMarkerPogress` is called with a status of complete for `profile-photo-modal`.

- **Profile feed starts rendering. During, the modal is clicked and starts rendering. Modal completes first.**

  1. `profile-photo` completes. `onMarkerProgress` is called with a status of complete for `profile-photo`. `onMarkerProgress` is also called on `profile` and `profile-feed` but both have pending unnamed suspense boundary, so they don’t complete.
  2. Modal is clicked and starts rendering.
  3. `profile-photo-modal` completes. `onMarkerPogress` is called with a status of complete for `profile-photo-modal`. `onMarkerProgress` is also called for `profile` and `profile-feed`, but both of them have a pending unammed suspense boundary, so they don’t complete.
  4. `profile-feed` completes. `onMarkerProgress` is called with a status of complete for `profile-feed` and `profile` because there are no more pending boundaries.

- **Profile feed and modal both start rendering. Child tracing marker of ProfilePhotoModal gets removed.**

  1. `onMarkerIncomplete` will be called for `ProfilePhotoModal` with a status of incomplete.
  2. The tracing markers in the `Profile` page will complete as normal.

# Drawbacks

- Introducing a new API adds more surface area and requires more to learn
- This API is meant as a low level implementation, and if developers want to ignore certain Suspense boundaries when trying to calculate if a transition is complete is fairly complicated
- Putting the transition callbacks at the root is complicated because you can have multiple concurrent interactions at the same time that only apply to a subtree, and it might be useful to pair the interaction roots at the Profiler level instead of at the root level.

# Alternatives

- Solutions that use ref counting to measure the duration from the moment users perform an interaction (ex. initial load/click/navigation) to when the results are shown on the screen.
- We previously created an Interaction Tracing API but removed it (see above for why).

# Adoption strategy

Both the `startTransition` and `createRoot` config options for transition tracing will be optional. Developers can opt into using transition tracing by adding the config options gradually if desired.

This feature will be adopted by React DevTools

# How we teach this

Document the new API and the new React DevTools features that follow.

# Unresolved questions

- **How should we tracking loading states that don’t use Suspense?**. We believe most use cases can be covered by Suspense. For the cases that can’t, we believe that this can initially be implemented in userland with Suspense and throwing Promises. However, we acknowledge that there is likely a need for a first class implementation (ex. often product code suspense incorrectly, suspending for very localized loading states isn't a good pattern, some code is not yet using Suspense), so we plan to add this at some later point.
- **How should we implement hydration?** We only plan to support tracing from server start to all HTML loading on the page for V1, but we want to expand to support hydration as well later.
- **Should we add a Suspense boundary ignore list on `startTransition`?** There may be potential edge cases where using the callbacks alone will not be sufficent to properly ignore a subtree. We anticipate that, because of the way React is architected, that developers won’t realistically encounter this issue. However, if they do, we will add a ignore list to the startTransition config that gives developers the option to ignore subtrees of named suspense boundaries.
- **How do we tracing updates that aren’t initiated from a transition?** Because we expect all updates that have the potential of suspending to be wrapped in a transition, we can technically build tracing these updates in userland. However, in service of a cohesive API, we are considering providing an API to do this.
- **Should we add the option to add transitionCallbacks to the Profiler component as well as the root?** Often interactions apply to a subtree rather than the whole tree. For example, in a data notebook that is currently refreshing 3 cells as separate interactions, reloading a cluster, and showing a modal, we have five different subtrees that are doing their own non-conflicting interactions. These can all be traced by the parent, but if these updates all get batched together it would require a lot of work on the developer's side to figure out which interactions have finished. We could add `transitionCallbacks` to the Profiler component as well to simplify things.
- **What should cause a transition to be marked as incomplete?** Are the hueristics we have currently to determine whether an interaction is incomplete the correct ones?
