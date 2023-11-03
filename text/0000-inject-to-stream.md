- Start Date: 2022-04-24
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Today, library authors cannot inject chunks into the React SSR stream in a way that works across all SSR frameworks (Next.js, Hydrogen, vite-plugin-ssr, ...).

The problem is that it is the SSR framework that integrates the React SSR stream. This means that there is no way for a library author to access the React SSR stream (without cooperation from each SSR framework).

This RFC proposes a new React API `injectToStream(chunk: string)` (along with a new hook `useStream()`), so that library authors can use it to inject HTML chunks to the React SSR stream. The point here is that `injectToStream()` works accross all SSR frameworks.

Note that the current situation is highly problematic. Without something like `injectToStream()`, library authors cannot leverage React SSR streaming: we cannot expect each library author to coordinate with each SSR framework.

That's why I believe this topic to be of high importance and high urgency.

# Basic example

```jsx
// New `useStream()` hook
import { useStream } from 'react'

function SomeComponent() {
  const stream = useStream()
  if (stream === null) {
    // No stream available. (Client-side, or when there isn't any SSR stream at all.)
    // ...
  }
  const { injectToStream } = stream
  injectToStream('<script type="application/json">[{"some":"data"}]</script>')
  // ...
}
```

# Motivation

I'm the author of an RPC tool [Telefunc](https://telefunc.com/) and I'm working on integrating it with the React SSR stream, so that the user can use Telefunc to fetch data:

```jsx
// TodoList.jsx
// Environment: Browser & Node.js server (SSR)

// Telefunc transforms `TodoList.telefunc.js` into a thin HTTP client.
import { fetchTodoItems } from './TodoList.telefunc'
import { useTelefunc } from 'telefunc'

function TodoList() {
  const todoItems = useTelefunc(async () => {
    const todoItems = await fetchTodoItems()
    return todoItems
  })
  return (
    <ul>
      { todoItems.map(todoItem =>
        <li>{todoItem.text}</li>
      )}
    </ul>
  )
}
```

```jsx
// TodoList.telefunc.js
// Environment: Node.js server

export async function fetchTodoItems() {
  // This works because `fetchTodoItems()` is always run on the server-side. (Telefunc makes
  // an HTTP request when `fetchTodoItems()` is called remotely from the client-side.)
  const todoItems = await query("SELECT text FROM todo_items;")
  return todoItems
}
```

The current working prototype uses [react-streaming](https://github.com/brillout/react-streaming).

For users that manually integrate React into their stacks (e.g. with [vite-plugin-ssr](https://vite-plugin-ssr.com/)), Telefunc can require the user to use `react-streaming`. So that Telefunc can access the SSR stream over `react-streaming`.

But, with other SSR frameworks such as Next.js, it not the user but Next.js that integrates the React SSR stream. This means that there is no way for Telefunc to access the SSR stream.

The same goes for all others React frameworks (Hydrogen, Remix, etc.).


# Detailed design

This RFC proposes a new API `injectToStream()` with two ways to access it:
 1. With `useStream()`:
    ```jsx
    import { useStream } from 'react'

    function SomeComponent() {
      const stream = useStream()
      if (stream === null) {
        // No stream available.
      }
      const { injectToStream } = stream
    }
    ```
    Enabling all kinds of higher-level hooks, such as [react-streaming](https://github.com/brillout/react-streaming)'s `useSsrData()` and `useAsync()`, and Telefunc's `useTelefunc()`.
 1. With `renderToPipeableStream()` (or `renderToReadableStream()`):
    ```jsx
    import { renderToPipeableStream } from 'react-dom/server'
    const { pipe, injectToStream } = renderToPipeableStream(<App />)
    ```
    Enabling SSR frameworks such as Next.js to inject scripts to the HTML.

For more details see [`react-streaming`'s implementation of `injectToStream()`](https://github.com/brillout/react-streaming/blob/e90e29690d1029050ec06f923877fdaaa1209f5d/src/renderToStream.ts).

# Drawbacks

None AFAICT.

# Alternatives

> What other designs have been considered?

Higher-level hooks such as [react-streaming](https://github.com/brillout/react-streaming)'s `useSsrData()` and `useAsync()`.

But `injectToStream()` is lower-level and fundamentally more flexible. Libraries like `react-streaming` can leverage `injectToStream()` to implement and provide higher-level hooks. (This is actually already the case: `useSsrData()` and `useAsync()` are based on top of `injectToStream()`.)

Library authors (Telefunc, React Query, CSS-in-JS tools, ...) can then use the higher-level hooks provided by tools like `react-streaming`.

That said, I think it could make sense to make `useAsync()` a React built-in hook, but this would be another (low-priority & non-urgent) RFC.

> What is the impact of not doing this?

Without something like `injectToStream()`, library authors cannot leverage React 18's new SSR streaming capabilities.

Not only Telefunc, but also React Query, GraphQL libaries, CSS-in-JS libraries, etc.

# Adoption strategy

I believe this can be released without any breaking changes.

# How we teach this

Simply adding `injectToStream()` to the docs will probably be enough.

# Unresolved questions

From a library author perspective none, AFAICT.
