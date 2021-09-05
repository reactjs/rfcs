- Start Date: 2021-09-05
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Cancel outdated async effects.

# Basic example

Basic solution: Manually cancel with `AbortSignal`.

```js
import { useEffect, useState } from 'react'

function Component(props) {
    const [loading, setLoading] = useState(false)
    const [text, setText] = useState("")
    useEffect(async signal => {
        setLoading(true)
        const data = await getData(props.id)
        if (signal.aborted) return
        setText(data)
    }, [props.id])
    return <>Some JSX</>
}
```

Advanced solution: Automatic cancel with [co](https://www.npmjs.com/package/co)-like API

```js
useEffect(function* (_signal) {
    setLoading(true)
    // React automatically stops if the deps array outdated.
    setText(yield getData(props.id))
}, [props.id])
```

# Motivation

## Cancel the native effects

Many effects in Web API now supports cancelling with `AbortSignal`, for example, `addEventListener` or `fetch`.
Providing an `AbortSignal` helps it easier to write.

```js
useEffect(signal => document.addEventListener('sth', () => something(), { signal }))
```

## Cancel custom effects

Setting state or refs need to be very careful otherwise it might cause race conditions.
Consider the following code:

```js
import { useEffect, useState } from 'react'

function Component(props) {
    const [loading, setLoading] = useState(false)
    const [text, setText] = useState("")
    useEffect(() => {
        setLoading(true)
        getData(props.id).then(setText)
    }, [props.id])
    return <>Some JSX</>
}
```

If the `getData` of a new render is resolved **eariler** than the old render, `text` will be in a bad state.

# Detailed design

Possible solution:

- Provide `signal` as the first parameter when calling `useEffect` callback.
- Accept generator functions so that React can take charge of how async things runs inside the effect.

# Drawbacks

Why should we *not* do this? Please consider:

- Can be a userland library.
    - But I think we should encourage to check if the signal expires before changing the state.
- Performance cost for creating so many AbortController?
- No need with Concurrent rendering?
    - I doubt it will be hard to design/use a generic (not only supporting GraphQL, fetch, etc...) concurrent-compatible async solution.
    - Community migration to concurrent rendering will be slow.
- Breaking change? One more argument provided.
- Generators compiled to ES5 is a very large state machine
    - BTW I ship modern ES sytnaxes to the user, so not a problem to myself.
- Many AbortError will appear in the console, it's annoying

# Alternatives

I don't know how to do this in concurrent rendering since we don't have a recommended way to write code in concurrent rendering.

# Adoption strategy

N/A.

# How we teach this

For `signal`, MDN is enough.

For automated cancellation of async work using generators, tell the developer just replace `await` with `yield`.

# Unresolved questions
