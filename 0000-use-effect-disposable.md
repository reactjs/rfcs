- Start Date: 2026-05-29
- RFC PR:
- React Issue:

# Summary

Allow returning a disposable from `useEffect()` and `useLayoutEffect()` instead of a callback function.

# Basic example

```jsx
import { useEffect } from 'react'

function MyComponent() {
  useEffect(() => {
    return {
      [Symbol.dispose]() {
        console.log('Disposed!')
      }
    }
  }, [])
}
```

# Motivation

JavaScript added support for [resource management](https://developer.mozilla.org/docs/Web/JavaScript/Guide/Resource_management). This allows JavaScript APIs to handle resource cleanup in a uniform way.

```js
using disposable = {
  [Symbol.dispose]() {
    console.log('Disposed!')
  }
}
```

React also supports disposable resources. In `useEffect()` and `useLayoutEffect()`, a user may return a cleanup function.

Since this is a JavaScript standard, we can expect more libraries and even browser builtins to use disposable resources.

# Detailed design

Currently the setup function of `useEffect()` and `useLayoutEffect()` must return a function or undefined. This proposal suggests to allow the setup to function to also return an object with a `Symbol.dispose` key instead. If the return value has `Symbol.dispose`, call that as a cleanup.

A practical use case would be something like this:

```jsx
import { useEffect } from 'react'

function MyComponent() {
  useEffect(() => {
    const disposer = new DisposableStack()
    const controller = disposer.adopt(new AbortController(), (c) => c.abort())

    fetch('https://example.com', { signal: controller.signal })
      .then((response) => response.text())
      .then(console.log, console.error)

    return disposer
  }, [])

  useEffect(() => {
    const handler = () => {
      console.log('Event fired')
    }

    const disposer = new DisposableStack()
    disposer.defer(() => window.removeEventListener('event', handler))

    window.addEventListener('event', handler)

    return disposer
  }. [])
}
```

But ideally, I hope something like this will be supported in the future:

```jsx
import { useEffect } from 'react'

function MyComponent() {
  useEffect(() => {
    const controller = new AbortController()

    fetch('https://example.com', { signal: controller.signal })
      .then((response) => response.text())
      .then(console.log, console.error)

    return controller
  }, [])

  useEffect(() => {
    return window.addEventListener('event', console.log)
  }. [])
}
```

# Drawbacks

- This adds a second way to handle `useEffect()` cleanups. [_“There should be one-- and preferably only one --obvious way to do it.”_](https://peps.python.org/pep-0020/#the-zen-of-python)
- At the moment of writing `Symbol.dispose` is [not yet available in all environments](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/Symbol/dispose#browser_compatibility).
- Users may be confused they should return a disposable, but not use the `using` keyword.

# Alternatives

N/A

# Adoption strategy

Nothing has to break. Developers may choose to return a disposable when the time is fit for them to do so.

# How we teach this

The terms _disposable_ and _disposer_ are pretty much used interchangeably. This is a JavaScript standard. No new terminology is introduced.

For teaching I believe it’s important to highlight that users should **not** use the `using` keyword to create their disposables. As far as the documentation goes, I don’t think much needs to change. It should only be stated disposables are supported.

# Unresolved questions

Should [`Symbol.asyncDispose`](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/Symbol/asyncDispose) be supported too?
