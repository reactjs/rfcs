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

This RFC describes several changes that we'd like to make to the behavior of the `<Suspense>` component:

- Behavior change: **Committed trees are always consistent**
- New feature: **Server-side rendering support with streaming**
- New feature: **Using transitions to avoid hiding existing content**
- Behavior change: **Layout effects re-run when content reappears**

The Suspense API itself does not change.

This RFC *does not* add support for data fetching. The changes in this RFC are prerequisites for that future step.

# Basic example

Suspense lets you declaratively specify the loading state for a part of the component tree if it's not yet ready to be displayed:

```js
<Suspense fallback={<Spinner />}>
  <Comments />
</Suspense>
```

Suspense makes the "UI loading state" a first-class declarative concept in the React programming model. This lets us build higher-level features on top of it. This RFC does not change the Suspense API itself, but it **refines its semantics and adds some new features** on top.

# Motivation

With the initial release in [React 16.6.0](https://reactjs.org/blog/2018/10/23/react-v-16-6.html), Suspense only supported a single use case: [code splitting on the client with `React.lazy`](https://github.com/reactjs/rfcs/pull/64). So even though you could add `<Suspense>` boundaries in your component trees, they were not being used by React for any other purposes. Moreover, you couldn't use them with server rendering. This made their usefulness very limited.

The full motivation has always been to extend the support so that eventually, the same declarative Suspense fallback can handle any asynchronous operations (loading code, data, images, etc). You can learn more about the full vision for Suspense in the [React 18 Keynote](https://youtu.be/FZ0cG47msEk?t=409).

This set of changes is the first step towards making Suspense more powerful.

# Detailed design

We'll first briefly recap how Suspense works, and then describe the changes.

## Recap: How Suspense works

Suspense lets you declaratively specify what React should show when a part of the tree is not yet ready to render:

```js
<Suspense fallback={<PageGlimmer />}>
  <RightColumn>
    <ProfileHeader />
  </RightColumn>
  <LeftColumn>
    <Suspense fallback={<LeftColumnGlimmer />}>
      <Comments />
      <Photos />
    </Suspense>
  </LeftColumn>
</Suspense>
```

Conceptually, you can think of `Suspense` as being similar to a `catch` block. However, instead of catching errors, it catches components "suspending". Any component in the tree can "suspend", which means that it's not ready to render. (The reason is arbitrary, but usually it could be due to missing code, data, etc.)

In JavaScript, when you `throw`, the closest `catch` above "wins", even if it's several function calls higher. Although Suspense works differently under the hood, the mental model is similar: if a component suspends, the closest `Suspense` component above the suspending component "catches" it, no matter how many components that are in between. In the above example, if `ProfileHeader ` suspends, then the entire page will be replaced with the `PageGlimmer`. However, if either `Comments` or `Photos` suspend, they together will be replaced with the `LeftColumnGlimmer`. This lets you safely add and remove Suspense boundaries according to the granularity of your visual UI design and without worrying which components exactly might depend on asynchronous code and data.

The exact mechanism of an arbitrary component "suspending" is out of scope of this RFC. The built-in `React.lazy` component suspends automatically if the code associated with the import has not yet loaded, and tells React to retry rendering when the code has loaded. We expect to add an API for an arbitrary component to suspend in a future RFC. Regardless of how the details of that API, for the purposes of this RFC, we can assume that any component might want to suspend and provide React with a Promise. React will *not* use the result of this Promise, but it will retry rendering. This can result in a completed render, an error, or getting suspended again (and displaying the fallback).

Importantly, Suspense is completely decoupled from *how* the code/data is being loaded. There are many possible strategies using different transport layers (e.g. GraphQL or REST), different places to fetch data (e.g. framework method vs ad-hoc), different performance characteristics (e.g. waterfall vs parallel pre-fetched), and different environments (e.g. client vs server). Suspense is only a mechanism to make React aware of the declarative loading states, and it does not prescribe any particular choice in how the data or code are fetched.

The changes in this RFC are independent of what suspends and why. They are focused on React's behavior after a component suspends.

## Behavior change: Committed trees are always consistent

Consider code like this:

```js
<div>
  {showComments && (
    <Suspense fallback={<Spinner />}>
      <Panel>
        <Comments />
      </Panel>
   </Suspense>
 )}
</div>
```

Suppose that `showComments` turns from `false` to `true`. React starts rendering the contents of the `Panel`, but `Comments` suspends. This means we can't show the `Panel` either: the entire contents up to the closest `Suspense` fallback needs to be hidden until the tree is ready. Only the `Spinner` should be visible until then.

Previously, React would do this in a sequence that goes like this:

1. Place `Panel` content into the DOM, but with a "hole" instead of `Comments` content.
1. Add `display: none` to the incomplete `Panel` content so that it does not appear visible.
1. Add the `Spinner` content into the DOM.
1. Fire the `Panel` effects because technically it has "mounted" (even though not fully).
1. Wait for the `Comments` to be ready.
1. Then, attempt rendering again.
1. Remove the `Spinner` content form the DOM.
1. Place the `Comments` content into the `Panel` content that was already in the DOM.
1. Remove `display: none` from the `Panel` content.

With this RFC, the proposed order is different:

1. **(New)** Throw away the `Panel` content instead of putting it into the DOM.
1. Add the `Spinner` content into the DOM.
1. Wait for the `Comments` to be ready.
1. Then, attempt rendering again.
1. Remove the `Spinner` content from the DOM.
1. Place the `Panel` content with `Comments` into the DOM.
1. **(Moved)** Fire the `Panel` effects.

This order is more intiutive because incomplete trees don't get committed at all. If a tree is not ready, it gets discarded, and a later attempt inserts a complete tree. Effects always observe a complete tree without "holes" in it.

The reason we didn't go with this approach in React 16.6 was for backwards compatibility reasons. At the time, most React code used classes, and many classes contained the `componentWillMount` method. This is why we renamed it to `UNSAFE_componentWillMount` in 2018 and [described](https://reactjs.org/blog/2018/03/27/update-on-async-rendering.html) strategies ot migrate away from it. The problem with `UNSAFE_componentWillMount` is that it fires during rendering (so, before we know whether child components suspended or not). If we throw away an incomplete tree after `UNSAFE_componentWillMount` has already fired, it will not receive a matching `componentDidMount` or `componentWillUnmount` call. (During a retry, there will be another `UNSAFE_componentWillMount` call because we need to render the same tree again.) So code that relies on `UNSAFE_componentWillMount` and `componentWillUnmount` being called the same number of times might cause mistakes or memory leaks.

We don't think this concern is relevant anymore for several reasons. `UNSAFE_componentWillMount` was marked as "unsafe" in 2018, and most popular open source libraries have long migrated away from it. This issue also only affects components "between" the `<Suspense>` node and the component that actually suspends. So it is very local in scope, and is easy to fix. Finally, Hooks have become a popular alternative to classes, and don't have the same issue. On the other hand, the current behavior [has been causing issues](https://github.com/facebook/react/issues/14536) for using popular component libraries with Suspense. This is why we think now is a good idea to make that change.

Read https://github.com/reactwg/react-18/discussions/7 for more details on the proposed new behavior.

## New feature: Server-side rendering support with streaming

Previously, if a component suspends during server rendering, React would throw a hard error. In practice, it meant that apps using server rendering (or built with an SSR framework like Next.js or Remix) could not use Suspense for code splitting in a supported way.

We are adding a new server renderer that supports streaming HTML out-of-order. Unlike the old server renderer that synchronously produces a string, the new server renderer produces a stream. That stream starts with the initial HTML that can be flushed early. However, the new renderer is also fully integrated with Suspense, which means that it's able to "wait" for parts of the tree that are not ready, and emit fallback HTML (e.g. spinners) for them. When the content is ready, React emits the content HTML in the same stream along with a small inline `<script>` to insert it in the right place in the original DOM structure. As a result, even if some part of the page is slow on the server, the user sees a progressively loading page with all intentionally designed intermediate loading states — even before client JS loads.

This feature will be most useful when Suspense supports data fetching, as it will unlock [streaming HTML while waiting for data](https://github.com/facebook/react/issues/1739). That is not a part of the current RFC. However, even before that part is added, `<Suspense>` offers benefits for server rendering. In particular, `<Suspense>` is integrated with hydration. For example, if a `lazy` component has not loaded the code yet, but it's wrapped in `<Suspense>`, React is able to hydrate the rest of the app without waiting for the code-split chunk. React will preserve the *content* HTML it has received from the server, and then hydrate it after the corresponding client code has loaded. This can significantly improve performance because hydration no longer needs to wait for all the code-split chunks to finish loading. You can start hydrating as soon as the main bundle is ready.

Check https://github.com/reactwg/react-18/discussions/22 for more details on the new streaming API. You can also read https://github.com/reactwg/react-18/discussions/37 for a deep dive on how exactly the proposed architecture works, and what it enables. You can also [watch this talk](https://www.youtube.com/watch?v=pj5N-Khihgc) for a high-level overview of the new architecture.

The downside of this is that to take full advantage of streaming, ecosystem libraries that assume synchronous SSR render today might need to find different approaches. We've published some information for frameworks (https://github.com/reactwg/react-18/discussions/114), CSS-in-JS libraries using `<style>` tags (https://github.com/reactwg/react-18/discussions/110), and CSS libraries using `<link>` (https://github.com/reactwg/react-18/discussions/108). We don't yet have all the answers to how data fetching will work with SSR, and this is out of scope of this RFC.

## New feature: Using transitions to avoid hiding existing content

Any component may suspend as a result of rendering. This can happen for a component that was already shown to the user. In order for screen content to always be consistent, if an already shown component suspends, React has to hide its tree up to the closest `<Suspense>` boundary. However, from the user's perspective, this can be disorienting. Consider this tab switcher:

```js
function handleClick() {
  setTab('comments');
}

<Suspense fallback={<Spinner />}>
  {tab === 'photos' ? <Photos /> : <Comments />}
</Suspense>
```

In this example, if `tab` gets set to from `'photos'` to `'comments'`, but `Comments` suspends, the user will see a spinner. This makes sense because the user no longer wants to see `Photos`, the `Comments` is not ready to render anything, and React needs to keep the user experience consistent so it has no choice but to show the `Spinner` above.

However, sometimes this user experience is not desirable. In particular, it is sometimes better to show the "old" UI while the new UI is being prepared. You can use the new [`startTransition`](https://github.com/reactwg/react-18/discussions/41) API to make React do this:

```js
function handleClick() {
  startTransition(() => {
    setTab('comments');
  });
}
```

Here, you tell React that setting `tab` to `'comments'` is not an "urgent" update, but is a "transition" that may take some time. React will then keep the old UI in place and interactive, and will switch to showing `<Comments />` when it is ready.

### Providing immediate feedback

Doing something asynchronous without any feedback may also be confusing. For this reason, React will provide a `useTransition` Hook which returns a tuple like `[isPending, startTransition]`. You can then use `isPending` to reflect to the user that something is happening. The UI stays completely interactive — for example, the user is able to switch back to the `'photos'` tab if they'd like to.

```js
const [isPending, startTransition] = useTransition();

function handleClick() {
  startTransition(() => {
    setTab('comments');
  });
}

<Suspense fallback={<Spinner />}>
  <div style={{ opacity: isPending ? 0.8 : 1 }}>
    {tab === 'photos' ? <Photos /> : <Comments />}
  </div>
</Suspense>
```

In the above example, when you click, the user will still see `Photos` for a bit, but the parent `div` will have 0.8 opacity, signaling a transition.

### Avoiding waiting too long

Consider a situation where somebody adds a very slow data source somewhere inside `Photos` component:

```diff
function Photos() {
  return (
    <>
      <MyPhotos />
+     <TaggedPhotosVerySlow />
    </>
  )
}
```

Now a tab transition that would previously complete fast would get "stuck" for a longer period of time, blocking the user from seeing the already-completed `<MyPhotos />`! To fix this, you can add another `<Suspense>` boundary around the slow content:

```diff
function Photos() {
  return (
    <>
      <MyPhotos />
+      <Suspense fallback={<PhotosGlimmer />}>
        <TaggedPhotosVerySlow />
+      </Suspense>
    </>
  )
}
```

React transitions don't "wait" for the "new" Suspense boundaries added during that transition. Since the user hasn't seen this boundary's content before, showing its fallback immediately is not jarring. This lets the user see the *rest* of the content (such as `MyPhotos`) sooner. This means that moving `Suspense` nodes up and down the tree affects whether your transition is snappier (but shows more loading states) or more complete (and "waits" for more things to show them at once). There is no manual way to control the duration of the transition.

## Behavior change: Layout effects re-run when content reappears

Consider this example:

```js
function handleClick() {
  setTab('comments');
}

<Suspense fallback={<Spinner />}>
  <AutoSize>
    {tab === 'photos' ? <Photos /> : <Comments />}
  </AutoSize>
</Suspense>
```

Suppose that we don't want to use a transition, and want to show the spinner when switching tabs. Or maybe the developer has not yet had a chance to add the transition. In this case, React has to hide the content and show the fallback, and later toggle them back.

The problem with this is that the components inside the tree had no way to know that they were hidden (and shown later). For example, if the `AutoSize` component reads the DOM layout to determine its size and position, it will read `0` while it's hidden. It also wouldn't be notified that it's visible again when React makes it visible.

To solve this, this RFC proposes to run **layout effects only** on hide and show. Concretely, when React needs to hide the Suspense content, it will run the "cleanup" of the layout effects inside of that tree. When React is ready to show the Suspense content again, it will run the layout effects in that tree similar to when the tree first appeared. This way, as long as components like `AutoSize` contain layout-related logic in layout effects, they would usually "just work" with Suspense.

The downside of this design is that existing code assuming layout effects with `[]` only run "once" would not work correctly. However, this assumption is already violated by Fast Refresh, which is integrated in most popular toolsets and re-fires effects when saving a file. This assumption would also be violated by other planned features, such as a feature that lets you unmount a component while preserving its state. For this feature to work, it needs to be able to later run layout effects "again" on top of the existing state. In general, it's helpful to think of effects not as states of lifecycle of a component ("mounting", "updating" and "unmounting") but as isolated units of behavior which different features (like Suspense) may need to toggle. In practice, we've found that incompatible code is mostly found in libraries (rather than user code), and that the new [Strict Effects behavior of Strict Mode](https://github.com/reactwg/react-18/discussions/19) helps find these issues early. In either case, this only affects code that (1) uses Suspense, (2) uses layout effects, and (3) “resuspends”, which is unusual for the existing code splitting use case, and (4) already likely suffers from the "measurement while hidden" flaw described above. This is why we think this change is net positive.

See also https://github.com/reactwg/react-18/discussions/31.

# Drawbacks

The individual drawbacks of each of these changes are described inline in the corresponding sections above.

# Alternatives

- Leave Suspense to only work on the client, and not work for server rendering.
- Leave Suspense to only work for code splitting, and not work on generalizing it.
- Leave Suspense unable to express common patterns like showing "old UI" while "new UI" is prepared.
- Leave third-party component authors without a way to reliably handle the cases where content is hidden or shown.
- Add more targeted Hooks for React just for handling Suspense states. (We didn't go that way because we're going to need similar constraints for other planned features anyway.)
- Make transitions always wait for all content to be ready. (We've started with this, and it turned out to make it too easy to introduce significant performance regressions to existing transitions by adding something deep in the tree.)
- Provide a way to configure how long a transition should take before showing the fallback. (We've started with this, and it turned out there is usually no meaningful number one can pick anyway, and the "new vs existing" boundary heuristic works better in practice.)
- Make Suspense run effects early and commit inconsistent trees to the DOM. (That's how it currently works, and it's causing issues.)
- Manage all loading states manually, and solve these problems outside of React. (We think React has the most leverage to solve these problems. The way the new SSR architecture mostly "just works" with the existing `Suspense` boundaries is a testament to that.)

# Adoption strategy

We plan to ship the new `Suspense` behavior changes as part of React 18. As you upgrade to React 18 and [switch](https://github.com/reactwg/react-18/discussions/5) from `ReactDOM.render` to `ReactDOM.createRoot`, you will get the new behavior. Similarly, when you [switch](https://github.com/reactwg/react-18/discussions/22) from `ReactDOMServer.renderToString` to `ReactDOMServer.renderToPipeableStream`, you will get the new streaming behavior. If you use a framework, these changes will likely happen under the hood as you upgrade to React 18 and your frameworks upgrades to be compatible with it.

Although the changes are described above in detail, in practice we've been able to roll them out across a massive codebase without significant problems. In the few cases where there were issues, they were usually at a library level and fixable with a few days of effort.

# How we teach this

We will update the API reference to describe the nuances of the new behavior. Most of these changes are behind-the-scenes, so the main user-facing feature is how Suspense interacts with `startTransition`. As Suspense becomes more useful (in particular, after its data fetching story is ready), it will take a more prominent place in our tutorials, along with guides focused on how to place the Suspense boundaries well.

# Unresolved questions

This RFC encompasses a few years of research, experimentation, development, and iteration, so it's mostly self-contained.

There are, however, many open questions that are outside the scope of this particular RFC:

- The exact protocol a component should use to tell React that it's suspended.
- How data fetching frameworks can integrate with Suspense.
- How to port every existing pattern to the streaming SSR.
- More control over when and how the fallbacks are shown.
- Animating from fallbacks to content and back.

We hope to have more clarity on these topics in the future.
