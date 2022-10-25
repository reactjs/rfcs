- Start Date: 2022-10-13
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Adds first class support for reading the result of a JavaScript Promise using Suspense:

- Introduces support for async/await in Server Components. Write Server Components using standard JavaScript `await` syntax by defining your component as an async function.
- Introduces the `use` Hook. Like `await`, `use` unwraps the value of a promise, but it can be used inside normal components and Hooks, including on the client.

This enables React developers to access arbitrary asynchronous data sources with Suspense via a stable API.

This proposal is not a complete data fetching solution on its own because it does not address caching. `await` and `use` are primitives for reading the asynchronous result of a promise, but they do not specify the lifetime of the underlying data. While existing data libraries that implement a caching strategy should find it easy to integrate without significant changes, a separate RFC will introduce an API called `cache` to assist with caching data. Taken together, these proposals will unlock official support for Suspense for Data Fetching.

> Note: This RFC assumes familiarity with the original RFC for Server Components, and should be viewed as an extension/modification of that proposal. Likewise, it assumes familiarity with React Suspense.

- [Summary](#summary)
- [Basic examples](#basic-examples)
  - [Example: `await` in Server Components](#example-await-in-server-components)
  - [Example: `use` in Client Components and Hooks](#example-use-in-client-components-and-hooks)
- [Motivation](#motivation)
  - [Seamless integration with JavaScript ecosystem](#seamless-integration-with-javascript-ecosystem)
  - [Avoiding an uncanny valley between server and client](#avoiding-an-uncanny-valley-between-server-and-client)
  - [Discourage unnecessary coupling between fetching and reading](#discourage-unnecessary-coupling-between-fetching-and-reading)
  - [Shift complexity into React without being too prescriptive](#shift-complexity-into-react-without-being-too-prescriptive)
  - [Enable compiler-driven optimizations](#enable-compiler-driven-optimizations)
- [Detailed design](#detailed-design)
  - [Async Server Components](#async-server-components)
    - [Async Server Components cannot contain Hooks](#async-server-components-cannot-contain-hooks)
  - [`use(promise)`](#usepromise)
    - [Resuming a suspended component by replaying its execution](#resuming-a-suspended-component-by-replaying-its-execution)
    - [Reading the result of a promise during a replay](#reading-the-result-of-a-promise-during-a-replay)
    - [Reading the result of a promise that was read previously](#reading-the-result-of-a-promise-that-was-read-previously)
    - [Reading the result of a promise during an unrelated update](#reading-the-result-of-a-promise-during-an-unrelated-update)
    - [Caveat: Data requests must be cached between replays](#caveat-data-requests-must-be-cached-between-replays)
    - [Conditionally suspending on data](#conditionally-suspending-on-data)
    - [Passing a promise from a Server Component to a Client Component](#passing-a-promise-from-a-server-component-to-a-client-component)
    - [Other "Usable" types](#other-usable-types)
- [Frequently asked questions](#frequently-asked-questions)
  - [Why isn't `use` called something more specific?](#why-isnt-use-called-something-more-specific)
  - [Why can't `use` be called in regular, non-React functions?](#why-cant-use-be-called-in-regular-non-react-functions)
  - [How about calling it [alternate name] instead?](#how-about-calling-it-alternate-name-instead)
  - [Why can't Client Components be async functions?](#why-cant-client-components-be-async-functions)
  - [Why not generator functions?](#why-not-generator-functions)
- [Unresolved questions](#unresolved-questions)

# Basic examples

## Example: `await` in Server Components

Server Components can use standard async/await syntax to access promise-based APIs. No special bindings are required:

```js
// This example was adapted from the original Server Components RFC:
// https://github.com/reactjs/rfcs/pull/188
async function Note({id, isEditing}) {
  const note = await db.posts.get(id);
  return (
    <div>
      <h1>{note.title}</h1>
      <section>{note.body}</section>
      {isEditing ? <NoteEditor note={note} /> : null}
    </div>
  );
}
```

This is the recommended way to access asynchronous data on the server.

A limitation of async Server Components is that they cannot access Hooks. We don't expect this to be an issue because Server Components are stateless and Hooks are rarely useful in that environment. However, you can use supported Hooks (for example, `useId`) inside Server Components that are written as regular functions.

## Example: `use` in Client Components and Hooks

`await` is not supported inside components that run on the client, due to technical limitations that will be explained in a later section. Instead, React provides a special Hook called `use`. You can think of `use` as a React-only version of `await`. Just as `await` can only be used inside async functions, `use` can only be used inside React components and Hooks:

```js
// `use` inside of a React component or Hook...
const data = use(promise);

// ...roughly equates to `await` in an async function
const data = await promise;
```

`use` has a special ability that other Hooks do not — calls can be nested inside conditions, blocks, and loops. This allows you to conditionally wait for data to load without splitting your logic into separate components:

```js
function Note({id, shouldIncludeAuthor}) {
  const note = use(fetchNote(id));

  let byline = null;
  if (shouldIncludeAuthor) {
    const author = use(fetchNoteAuthor(note.authorId));
    byline = <h2>{author.displayName}</h2>;
  }

  return (
    <div>
      <h1>{note.title}</h1>
      {byline}
      <section>{note.body}</section>
    </div>
  );
}
```

Promises are not the only type that can be unwrapped with `use` — [additional "usable" types will be supported](#other-usable-types), including Context.

# Motivation

## Seamless integration with JavaScript ecosystem

One of our primary motivations is to provide a seamless integration with the rest of the JavaScript ecosystem, where promises are the idiomatic way to represent asynchronous values.

However, the original proposal for Server Components did not make it easy to access promise-based APIs. Each data source needed to be wrapped with special bindings that were tricky to implement correctly. There was no straightforward way to wrap an arbitrary, one-off async operation.

We were frequently asked: why not use async/await? Originally, we wanted to have a consistent API for accessing data across Server Components, Client Components, and Shared Components. However, we're unable to support async/await in Client Components for technical reasons we'll cover in a later section.

Ideally we would prefer to support async/await everywhere. We have no desire to replace promises with our own asynchronous primitive, nor do we want to require every asynchronous API to be wrapped React-specific bindings. But at the time of the original RFC, we judged that having a consistent API across Server and Client Components was the more important goal.

We've since changed our mind. We now believe that the benefits of async/await in Server Components outweigh the downsides of having a different API (`use`) on the client.

One reason is that we underestimated how frequently Server Components might need to perform ad hoc asynchronous operations. Promise-based APIs are ubiquitous in JavaScript server applications, and it's annoying (if not completely infeasible) to have to write React-specific bindings for every one. This was one of the most frequent criticisms we received about the original proposal.

We realized that in our desire to provide a consistent API across Server and Client components, we hadn't actually removed the need for async/await everywhere else — developers would still need to use async/await when performing logic outside React components, like submitting forms and processing server responses.

## Avoiding an uncanny valley between server and client

Despite our initial hesitance, there is also an advantage to having different ways of accessing data on the server versus the client: it makes it easier to keep track of which environment they're working in.

Server Components are meant to feel similar to Client Components, but we don't want them to feel _too_ similar. Each environment has different capabilities, and the distinction needs to be clear to developers as they structure their applications. For example, it's generally best to do as much as possible in Server Components, and only extract the minimal amount to Client Components when necessary.

Making Server and Client components easy to distinguish at a glance will help avoid an uncanny valley, where developers must constantly spend mental energy discerning which components run in which environment.

Async functions provide a clear visual signal: if a component is written as an async function, that's a Server Component.

Even in the future, if we were able add support for async components on client, we would likely continue to recommend a separation between async components (i.e. those that fetch data) and stateful ones (i.e. those that use Hooks) to encourage easy refactoring. A common workflow might be to refactor a single component that both fetches data and uses state into multiple components that handle fetching and state separately — then, go one step further and move the fetching to the server.

## Discourage unnecessary coupling between fetching and reading

A nice property of both `await` and `use` is they have nothing to do with how the data for a promise is requested — they are only concerned with unwrapping async values, regardless of how they were fetched.

In previous proposals for Suspense-based data fetching APIs, because there was no built-in way to read an async value, it was often the case that fetching and rendering were unnecessarily coupled. The idea was that each data source should provide separate `preload` and `read` methods — one to fetch, one to unwrap — but this was purely conventional and there was no guarantee it would be consistent between libraries.

By establishing a standard way to read an async value, we hope to discourage unnecessary coupling between fetching and rendering.

For example, a common performance pattern is to optimistically prefetch data that might be needed in a future render, without blocking the current render. `use` makes this straightforward, regardless of which library you use to fetch the data:

```js
function TooltipContainer({showTooltip}) {
  // This is a non-blocking fetch. We initiated the request but we haven't
  // yet unwrapped the result.
  const promise = fetchInfo();

  if (!showTooltip) {
    // If `showTooltip` is false, we can return immediately without waiting
    // for the data to finish loading.
    return null;
  } else {
    // If `showTooltip` is true, we wait for the promise to resolve by passing
    // it to `use`. It probably already loaded, because the request was
    // initiated during the previous render.
    return <Tooltip content={use(promise)} />;
  }
}
```

## Shift complexity into React without being too prescriptive

In the past we've been reluctant to add an official API for data fetching because we didn't want to lock developers into a suboptimal architecture. Data fetching is a large problem space with many trade offs, and the solutions that work best are the ones that are deeply integrated into the rest of the application's architecture — for example, the router.

However, there is no single implementation of a React architecture, and the React ecosystem has benefited from innovation that happens in third-party libraries and frameworks. If React makes too many assumptions about how data fetching should work, it could inhibit the best solutions from emerging in userspace. On the other hand, if React makes too _few_ assumptions, we forfeit our ability to make across-the-board improvements for everyone.

The goal of this proposal is provide a shared set of primitives for data fetching that can be used across different React frameworks, without being too prescriptive about how data fetching should work under the hood. You should be able to recognize React code and patterns even when switching between frameworks — or when not using one. For example, Next.js and Remix — two popular and influential React frameworks — may have different strategies for routing or cache invalidation, but they don’t need to invent their own API for loading states, or conditional loading, or error handling. They can use the patterns that are built into React.

A related goal is to make it easier for developers to scale between simple and complex use cases. You should be able to incrementally upgrade your components to more advanced features as the requirements of your app become more complex. Similarly, if you’re already using a sophisticated data framework, you should be able to add simple, one-off data fetches to a single component without compromising the architecture of the overall system.

## Enable compiler-driven optimizations

Because async/await is a syntactical feature, it's a good candidate for compiler-driven optimizations. In the future, we could compile async Server Components to a lower-level, generator-like form to reduce the runtime overhead of things like microtasks, without affecting the outward behavior of the component. And although `use` is not technically a syntactical construct in JavaScript, it effectively acts as syntax in the context of React applications (we will use a linter to enforce correct usage) so we can apply similar compiler optimizations on the client, too.

We've taken care to consider this throughout the design. For example, in the current version of React, an unstable mechanism allows arbitrary functions to suspend during render by throwing a promise. We will be removing this in future releases in favor of `use`. This means only Hooks will be allowed to suspend. An [auto-memoizing compiler](https://reactjs.org/blog/2022/06/15/react-labs-what-we-have-been-working-on-june-2022.html#react-compiler) can take advantage of this knowledge to prevent arbitrary function calls from unnecessarily invalidating a memoized computation.

# Detailed design

This section will cover the technical details of both async Server Components and the `use` Hook.

Because Server Components are stateless and run in an asynchronous environment, the implementation of async Server Components is fairly straightforward. Most of the complexity in this proposal is related to reading promises inside Client Components, which may need to render synchronously in response to user input. This makes `use` tricky to implement given the runtime behavior of promises as imposed by the JavaScript specification.

We'll cover how async Server Components work first before diving into `use`.

## Async Server Components

From the perspective of the runtime, an async Server Component is defined as any Server Component that returns a promise. React will wait for the promise to resolve, then the fulfilled value be rendered the same as if it were returned directly.

In practice, we will encourage component authors to use async/await syntax rather than return a promise object directly, to help [avoid an uncanny valley between server and client components](#avoiding-an-uncanny-valley-between-server-and-client).

### Async Server Components cannot contain Hooks

> Note: Hooks are still permitted in Server Components that are defined as a regular, non-async function.

A constraint of async Server Components is that they cannot contain Hooks. This is partly motivated by technical limitations, but it's also an intentional design decision to help [distinguish between server and client components](#avoiding-an-uncanny-valley-between-server-and-client).

This may seem like an onerous restriction if you're used to writing Client Components, but in practice we don't expect this to be an issue because there are very few Hooks that are useful in Server Components (which run on the server only, or at build time). Stateful Hooks like `useState` aren't permitted regardless, and most of the rest (like `useMemo`) are really only available to increase the number of Shared Components that run in both server and client environments; they don't actually provide much functionality.

In general what we’ve found is that most use cases for hooks in Server Components can be replaced with a request-local version of the same API. For example, instead of a useRequestHeaders hook, frameworks can provide a getRequestHeaders function that reads from AsyncLocalStorage. It doesn’t need to be contextual per tree.

However, in the exceptional cases, Hooks are still permitted in Server Components that are defined as a regular, non-async function.

## `use(promise)`

`use` is designed to provide much the same programming model as async/await, while still working inside regular (non-async) function components and Hooks. Similar to async functions in JavaScript, the runtime for `use` maintains an internal state machine to suspend and resume; but from the perspective of the component author, it looks and feels like a sequential function:

```js
function Note({id}) {
  // This fetches a note asynchronously, but to the component author it looks
  // like a synchronous operation.
  const note = use(fetchNote(id));
  return (
    <div>
      <h1>{note.title}</h1>
      <section>{note.body}</section>
    </div>
  );
}
```

According to the JavaScript spec, the resolved value of a promise is always fulfilled (or rejected) asynchronously. There's no way to synchronously inspect its value, even if the underlying data has already finished loading. This was an intentional design decision by the authors of the language, to avoid data races caused by ambiguous sequencing.

While the motivation behind this design is understandable, it presents a problem for React and React-like UI libraries, which model UI as a function of props and state at a given moment. This is a suprisingly challenging problem to solve without support from the language. There are a few different strategies React can use, depending on the situation.

### Resuming a suspended component by replaying its execution

If a promise passed to `use` hasn't finished loading, `use` suspends the component's execution by throwing an exception. When the promise finally resolves, React will _replay_ the component's render. During this subsequent attempt, the `use` call will return the fulfilled value of the promise.

Unlike async/await or a generator function, a suspended component does not resume from the same point that it last left off — the runtime must re-execute all of the code in between the beginning of the component and where it suspended. Replaying relies on the property that React components are required to be idempotent — they contain no external side effects during rendering, and return the same output for a given set of inputs (props, state, and context). As a performance optimization, the React runtime can memoize some of this computation. It's conceptually similar to how components are reexecuted during a state update.

When resuming a component that previously suspended, `use`'s reliance on replaying has additional runtime overhead compared to async/await. However, if the data for a component has already resolved — like if the the data was preloaded, or during an unrelated re-render — `use` actually has less overhead compared to async/await because it can unwrap the resolved value without waiting for the microtask queue to flush.

### Reading the result of a promise during a replay

When replaying a suspended component after the promise resolves, eventually React will reach the same `use` call that suspended during the previous attempt. Although this is conceptually the same call, the promise passed to `use` may or may not be the same promise instance that was passed during the last attempt. However, regardless of whether the instances are identical, we can assume that they represent the same result, as long as none of the props or state have changed in the meantime. React will reuse the result from the previous attempt, and ignore the promise that was created during the replay. This once again takes advantage of the fact that React components are idempotent functions.

### Reading the result of a promise that was read previously

If props or state have changed, React can't assume that a promise passed to `use` has the same result as the previous attempt. We need a different strategy.

The first thing React will try is to check if the promise was read previously, either by a different `use` call or a different render attempt. If so, React can reuse the result from last time, synchronously, without suspending.

React does this by adding additional properties to the promise object:

- The **`status`** field is set to one of `"pending"`, `"fulfilled"`, or `"rejected"`.
- When a promise is fulfilled, its **`value`** field is set to the fulfilled value.
- When a promise is rejected, its **`reason`** field is set to the rejection reason (which in practice is usually an error object).

(The naming of these fields is borrowed from [`Promise.allSettled`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/allSettled).)

Keep in mind that React will not add these extra fields to _every_ promise, only those promise objects that are passed to `use`. It does not require modifying the global environment or the Promise prototype, and will not affect non-React code.

Although this convention is not part of the JavaScript specification, we think it's a reasonable way to track a promise's result. The ideal is that the lifetime of the resolved value corresponds to the lifetime of the promise object. The most straightforward way to implement this is by adding a property directly to the promise.

An alternative would be to use a WeakMap, which offers similar benefits. The advantage of using a property instead of a WeakMap is that other frameworks besides React can access these fields, too. For example, a data framework can set the `status` and `value` fields on a promise preemptively, before passing to React, so that React can unwrap it without waiting a microtask.

If JavaScript were to ever adopt a standard API for synchronously inspecting the value of a promise, we would switch to that instead. (Indeed, if an API like `Promise.inspect` existed, this RFC would be significantly shorter.)

### Reading the result of a promise during an unrelated update

Tracking the result on the promise object only works if the promise object did not change between renders. If it's a brand new promise, then the previous strategies won't work. However, in many cases, even a brand new promise will already have resolved data. This happens often because most promise-based APIs return a fresh promise instance on every call regardless of whether the response was cached. That's also how async functions work in JavaScript — every call to an async function results in a brand new promise, even if the data was cached, and even if nothing was awaited at all.

The most important case where this happens is when an component re-renders in response to an unrelated update.

Consider this example:

```js
async function fetchTodo(id) {
  const data = await fetchDataFromCache(`/api/todos/${id}`);
  return {contents: data.contents};
}

function Todo({id, isSelected}) {
  const todo = use(fetchTodo(id));
  return (
    <div className={isSelected ? 'selected-todo' : 'normal-todo'}>
      {todo.contents}
    </div>
  );
}
```

If the `id` prop updates, it makes sense that `fetchTodo` might have to suspend until the new data has finished loading. But what if `isSelected` updates while `id` remains the same? The promise passed to `use` will be different, because async functions always return new promise instances. But we shouldn't have to suspend again, because the data was read from a cache.

If React didn't handle this case properly, arbitrary updates could cause the UI to suspend even when no new data has been requested. So we need some strategy to handle this.

Here's where our tricks start getting more complicated.

What we can do in this case is rely on the fact that the promise returned from `fetchTodo` will resolve in a microtask. Rather than suspend, React will wait for the microtask queue to flush. If the promise resolves within that period, React can immediately replay the component and resume rendering, without triggering a Suspense fallback. Otherwise, React must assume that fresh data was requested, and will suspend like normal.

This allows us to avoid suspending as long as the data requests are cached.

Note that it's still better for performance if the promise objects themselves are cached, because then we can read the result synchronously without waiting for microtasks and without replaying the component. So if this happens during a high priority update, such as in response to a user input event, React will log a performance warning. The way to fix the warning is to either memoize the async function (with `useMemo`) or cache it (with `cache`, which will be described in an upcoming RFC).

In the future, an auto-memoizing compiler could address this problem by memoizing promise objects automatically.

### Caveat: Data requests must be cached between replays

The mechanism of waiting for microtasks to flush before suspending only works if the data requests are cached. More precisely, the constraint is: an async function that re-renders without receiving new inputs must resolve within a microtask.

(Note that this caveat does not apply to async Server Components, because they do not receive updates.)

**This will be covered extensively in the upcoming RFC for the `cache` API.** It's very unlikely we will ship `use` without `cache`. The API will look something like this:

```js
// A cached function returns the same response for a given set of inputs —
// id, in this example. The `cache` proposal will also have a mechanism to
// invalidate the cache, and to scope it to a particular part of the UI.
const fetchNote = cache(async (id) => {
  const response = await fetch(`/api/notes/${id}`);
  return await response.json();
});

function Note({id}) {
  // The `fetchNote` call returns the same promise every time until a new id
  // is passed, or until the cache is refreshed.
  const note = use(fetchNote(id));
  return (
    <div>
      <h1>{note.title}</h1>
      <section>{note.body}</section>
    </div>
  );
}
```

We'll update this proposal once the RFC for `cache` is ready.

Note that most existing data fetching libraries already implement a caching mechanism that is sufficient to avoid this pitfall, so they should be able to adopt the `use` pattern without also needing to adapt `cache`. This is mostly a new concern that arises from calling async functions directly within a component.

### Conditionally suspending on data

Unlike all other Hooks, `use` can be called conditionally, rather than being limited to only the top level of the function. The motivation for this is to support conditionally suspending on data without needing to extract it to a separate component:

```js
function Note({id, shouldIncludeAuthor}) {
  const note = use(fetchNote(id));

  let byline = null;

  if (shouldIncludeAuthor) {
    // Because `use` is inside a conditional block, we avoid blocking
    // unncessarily when `shouldIncludeAuthor` is false.
    const author = use(fetchNoteAuthor(note.authorId));
    byline = <h2>{author.displayName}</h2>;
  }

  return (
    <div>
      <h1>{note.title}</h1>
      {byline}
      <section>{note.body}</section>
    </div>
  );
}
```

The rules regarding where `use` can be called in a React component correspond to where `await` can be invoked in an async function, or `yield` in a generator function. `use` can be called from within any control flow construct including blocks, switch statements, and loops. The only requirement is that the parent function must be a React component or Hook. So, for example, `use` can be called inside a for loop, but it cannot be called inside a closure passed to a `map` method call:

```js
function ItemsWithForLoop() {
  const items = [];
  for (const id of ids) {
    // ✅ This works! The parent function is a component.
    const data = use(fetchThing(id));
    items.push(<Item key={id} data={data} />);
  }
  return items;
}

function ItemsWithMap() {
  return ids.map((id) => {
    // ❌ The parent closure is not a component or Hook!
    // This will cause a compiler error.
    const data = use(fetchThing(id));
    return <Item key={id} data={data} />;
  });
}
```

The reason `use` is allowed to be called conditionally is that, unlike most other Hooks, it does not need to track state across updates. Whereas a Hook like `useState` needs to be executed in the same position during every render so that React can associate it with its previous state, `use` does not need to "store" any data that lives beyond a single render of component. Instead, the data for the promise is associated with the promise object itself.

### Passing a promise from a Server Component to a Client Component

A planned feature for Server Components is to support passing a promise as a prop to a Client Component. This is slightly outside the scope of this proposal, but it's worth mentioning because it highlights why being able to contionally invoke `use` is so valuable.

TODO: Example

### Other "Usable" types

It may be initially confusing for developers that `use` has the special ability to be called conditionally, unlike all other Hooks. We think the feature is useful enough that it's worth a potential learning curve, and that developers will adjust quickly.

To mitigate confusion, the intent is that `use` will be the _only_ Hook that will _ever_ support conditional execution — instead of having to learn about a handful of Hooks that are exempted from the typical rules, developers will only have to remember one.

Though it's not strictly within the scope of this proposal, `use` will eventually support additional types besides promises. For example, the first non-promise type to be supported will be Context — `use(Context)` will behave identically to `useContext(Context)`, except that it can be called conditionally.

You can think of a "Usable" type as a container for a value. Calling `use` unwraps it.

```js
const resolvedValue = use(promise);
const contextualValue = use(Context);

// Potential future Usable types
// (Purely hypothetical, not part of this proposal)
const currentState = use(store);
const latestValue = use(observable);
// ...and so on
```

# Frequently asked questions

## Why isn't `use` called something more specific?

Two reasons:

- Promises are not the only ["usable" type](#other-usable-types) — for example, you'll also be able to be `use(Context)`.
- `use` is a very special Hook because [it's allowed be called conditionally](#conditionally-suspending-on-data). The idea is to mitigate confusion by making it the _only_ conditional Hook. Instead of remembering a handful of different conditional Hooks, developers will only have to remember one.

## Why can't `use` be called in regular, non-React functions?

Even though `use` can be [called conditionally](#conditionally-suspending-on-data), it's still a Hook because it only works when React is rendering. So it can only be executed by a React component or Hook.

Theoretically, it will "work" in the runtime if you call `use` inside a function which itself is only called from inside a React component or Hook, but this will be treated as a compiler error.

If we allowed `use` to be called in regular functions, it would be up to the developer to keep track of whether it was being in the right context, since there's no way to enforce this in the type systems of today. That was one of the reasons we created the "use"- naming convention in the first place, to distinguish between React functions and non-React functions.

It would also make it harder for a memoizing compiler to re-use computations, because it would have to assume that _any_ arbitrary function might possibly suspend.

We could introduce a separate naming convention that's different from hooks, but it doesn't seem worth adding yet-another type of React function for only this case. In practice, we don't think it will be a big issue, in the same way that not being able to call Hooks conditionally isn't a big deal.

## How about calling it [alternate name] instead?

Alternate suggestions for names are appreciated. Some things to keep in mind, though.

The name should not only suggest "unwrapping", it should also indicate that it's a React-only function. An advantage of "use" is that it builds on the naming convention established by the other Hooks already. A different name would require developers to learn two special words instead of one.

Here's how we might teach this to beginners:

- Any function that starts with "use" is a Hook — it can only be called from a React function (a component or custom Hook)
- Except for the function called "use" (no extra characters), all other Hooks can only be called at the top level.

## Why can't Client Components be async functions?

We strongly considered supporting not only async Server Components, but async Client Components, too. It's technically possible, but there are enough pitfalls and caveats involved that, as of now, we aren't comfortable with the pattern as a general recommendation. The plan is to implement support for async Client Components in the runtime, but log a warning during development. The documentation will also discourage their use.

The main reason we're discouraging async Client Components is because it's too easy for a single prop to flow through the component and invalidate its memoization, triggering the [microtask dance described in an earlier section](#reading-the-result-of-a-promise-during-an-unrelated-update). It's not so much that it introduces new performance caveats, but it makes all the performance caveats described above much more likely.

There is one pattern where we think async Client Components make sense: if you structure them in such a way that they're guaranteed to only update during navigations. But the only realistic way to guarantee this in practice is by integrating support directly into your application's router. So it's unclear to what extent this pattern will even be documented. At least for now.

In the future, we could unlock general support for async Client Components by compiling them to a specialized, generator-like runtime.

## Why not generator functions?

The main disadvantage of generators is that you can't use them inside custom hooks without effectively requiring every single Hook to be a generator. That adds a lot of extra overhead in the runtime in the case where you aren't loading data, as well as lots of syntactical overhead for the developer.

One plan we have to address this is for an auto-memoizing compiler to reuse the computation from the previous replay attempt.

Another plan is to compile async/await syntax to a lower level generator-like form. That would address the performance concern, though not the hook abstraction problem. To solve the abstraction problem, a longer term idea is to introduce custom syntax for invoking a hook, and compile all React functions to a generator-like runtime.

# Unresolved questions

The main piece missing from this proposal is `cache`, a API for caching async requests. This will be addressed in a companion RFC soon, at which point we'll update this proposal to include references where appropriate.

`cache` isn't required for this proposal to be viable but they were designed with each other in mind, so they should really be considered together as a unit.
