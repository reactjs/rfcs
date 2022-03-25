- Start Date: 2022-03-25
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

This RFC describes a new recovery mechanism for server errors in React 18. It lets you recover from errors thrown on the server by adding `<Suspense>` boundaries.

When an error is thrown on the server, React will emit the fallback HTML from the server and then automatically retry rendering on the client. This will affect everything up to the closest `<Suspense>` boundary above. Similarly, when an error is thrown during hydration on the client, React will discard the server-rendered HTML and revert to a clean client render.

If the client render succeeds, React considers the original error as "recoverable", because it was not surfaced to the user. React will still log the errors, in both development and production, so your error reporting can pick them up.

Additionally, missing/extra nodes and text mismatches are now treated like errors instead of warnings. This means that React will no longer attempt to "patch up" individual nodes by inserting or deleting a node on the client in an attempt to match the server markup. Instead, React will revert to client rendering. This ensures the rendering result is consistent in case of a mismatch, which is important for correctness and security. Hydration errors should not be ignored and should be treated like errors by the developer.

# Basic example

Previously, if a component throws and you render on the server, `renderToString` would throw.

Now, you can wrap a part of your app in a Suspense boundary:

```js
<Suspense fallback={<Spinner />}>
  <Comments />
</Suspense>
```

If `Comments` or something inside of `Comments` throws on the server, React will include the `<Spinner />` HTML output in the server rendered HTML along with a special marker. On the client, during hydration, React will automatically retry rendering `<Comments />`.

Hydration mismatches will also use the same mechanism to retry a client client render.

# Motivation

There used to be no way to handle a rendering error on the server, so any component error was fatal and deopted the entire app to a full client render (or, depending on how your app is set up, would completely break it). On the client, errors are handled by error boundaries. But error boundaries don't work on the server because they rely on state. Although it's possible there will later be a separate error boundary API for the server, in the meantime, this change provides a natural way for the app to recover from error. Since server and client run the same code, it is possible that the client will be able to succeed (e.g. if the error was caused by some server-only problem).

As you adopt Suspense, you will naturally have loading states wrapping different parts of your app. `<Suspense>` lets us reuse these intentionally designed loading states while we don't know whether the client can recover or not. If there is an error but your app can recover on the client, from a user's perspective, it will appear as if a part of your app took slightly longer to load, but still had an expected intentionally designed visual state. However, if it fails on the client, React would throw (and let the error boundary show the error).

The most controversial part of this proposal is the change to treat hydration warnings the same way. There are a few reasons for this:

1. React can't reliably "patch up" a mismatch. In the most worst cases, a mismatch could lead to a [privacy or security hole](https://github.com/facebook/react/issues/23381#issuecomment-1065355030).
2. Our existing heuristics for "patching up" don't always work with the newly added support for lazy-loading SSR'd components.
3. We didn't have a mechanism for automatic but granular error recovery, but now we do.

# Detailed design

## Transferring errors from the server to client

On the server, React will emit a special marker if the contents of a Suspense boundary has failed on the server. The user will see the HTML of the closest Suspense fallback content instead of the failed content.

On the client, React will recognize a failed Suspense boundary, and schedule a retry to render its content on the client side. If rendering is successful, the result is displayed. If rendering throws again, React will treat it as a regular throw during render, which means it will be handled by the closest error boundary.

## Hydration mismatches

### Attribute mismatch

Attribute mismatch works the same way as before. React warns about it in development only but does not attempt to patch it up.

### Text content and missing/extra nodes mismatch

Previously, React would try to "patch up" the tree to match the client render. However, this can lead to [privacy and security holes](https://github.com/facebook/react/issues/23381#issuecomment-1065355030). The new behavior is to throw away the server HTML up to the closest `<Suspense>` boundary, and then synchronously do a clean re-render from that part. This can be a bit slower and can blow away some DOM state (like an input already typed into) but guarantees a consistent output. If there is no `<Suspense>` boundary above the error, React retries a clean client render from the root and discards all server HTML.

In some cases, text mismatches are impractical to avoid (e.g. timestamps). The existing `suppressHydrationWarning` prop lets you mark a node as having intentionally different text content on the server and on the client. Unfortunately, its naming becomes confusing because now it *does* have production behavior: it prevents the client recovery re-render. We will likely want to rename it in a future release.

>Note: You might want to add some granularity to the hydration mismatch recovery by adding more `<Suspense>` boundaries. However, sometimes you might not have good intentionally designed loading states, and might be tempted to just wrap subtrees into `<Suspense fallback={null}>`. This is not a good pattern because the same "empty hole" boundary will get used in other situations, like during error recovery or if you lazy-load components. In the future, we plan to add a type of Suspense boundaries that you can mark to only used as a "last resort". Until then, it is recommended to only use them when you have an intentional loading state like a spinner.

## Error reporting

Both `createRoot` and `hydrateRoot` accept a new `onRecoverableError` callback as an option.

```js
hydrateRoot(container, <App />, {
  onRecoverableError(error) {
    // ...
  }
});
```

You can use it to log events to an error reporting service. Recoverable errors are different from regular errors because they don't lead to a completely broken user experience:

* For `hydrateRoot`, a recoverable error is reported if there was an error on the server or if there was a hydration error (but the client has retried rendering).
* For `createRoot`, a recoverable error is reported when a component throws during a "recovery" synchronous update. This comes up when you use a concurrent feature like `startTransition` and a component throws. In this case, React synchronously retries rendering "just in case" because this often works around concurrency bugs with libraries that aren't compatible yet. If that render is successful, the error is reported as a recoverable one.

We will likely want to add more kinds of recoverable errors in the future. If you don't specify `onRecoverableError`, the default implementation calls [`reportError`](https://developer.mozilla.org/en-US/docs/Web/API/reportError) when available, falling back to `console.error` if not.

# Drawbacks

- If you don't fix hydration errors, you will have bigger performance regressions than before. People often struggle to debug them because React doesn't provide clear enough messages for hydration errors.
- It is confusing that `suppressHydrationWarning` now has production behavior.
- SSR apps today don't have any `<Suspense>` nodes so they will always retry client render from the root at first. This can be an unpleasant surprise.
- Adding `<Suspense>` boundaries requires intentional loading states. Adding `<Suspense>` boundaries without an intentional loading state is not good for other features that rely on `Suspense`, until we add a mechanism to mark some boundaries as being "last resort" boundaries.

# Alternatives

- Server errors fail the entire page.
- Error boundaries on the server. (But this doesn't solve hydration.)
- Different other heuristics for trying to "patch up" the content.
- Use a separate kind of boundary for client error recovery instead of relying on `<Suspense>`.
- Rename `suppressHydrationWarning` in the current release instead of a future one.

# Adoption strategy

The feature works out of the box, so changing the existing code is not necessary.

The initial release will need to be fast-followed by two features:

- Better hydration error messages (so that they're easier to fix)
- A way to mark some Suspense boundaries as "last resort" (so that you can add empty fallbacks where you don't have intentional loading states)

In general, this would be a similar effort to introducing error boundaries in React 16. However, for the most part, since this proposal relies on existing concepts, it will be more of a behind-the-scenes change than something we need to actively teach people about.

# How we teach this

Document the new behaviors. Make sure that common server rendering guides (including framework ones) are updated.

# Unresolved questions

- When/how do we name `suppressHydrationWarning`
- The API for marking Suspense boundaries as "last resort"