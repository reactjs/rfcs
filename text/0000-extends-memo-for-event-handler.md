- Start Date: 2023-02-08
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Extends `memo` function to support mutable event handlers.

# Basic example

You need not wrap any event handler by `useCallback` or `useEvent`.

```js
import OriginalSendButton from 'some-where';

let SendButton = OriginalSendButton;

// When you want to optimize `SendButton`, just memoization like:
SendButton = memo(SendButton, {events: true});

function Chat() {
  const [text, setText] = useState('');

  const onClick = () => {
    sendMessage(text);
  };

  return <SendButton onClick={onClick} />;
}
```

# Motivation

Make `memo` work for mutable event handlers.

It is similar with `useEvent` rfc [first motivation](https://github.com/reactjs/rfcs/blob/useevent/text/0000-useevent.md#reading-stateprops-in-event-handlers-breaks-optimizations).

Before `memo` will break by mutable event handlers.

So a lot of people wrap every event handler with `useCallback` or `useEvent`.
The wrapper is annoying, and most of the time it just makes performance worse.


# Detailed design

## Internal implementation

Internally, `memo` function will approximately work like this:

```jsx
// (!) Approximate behavior
import {memo as reactMemo, useRef} from 'react';

export function memo(Component, options) {
  if (!options || typeof options === 'function') {
    return reactMemo(Component, options);
  }

  const {events, propsAreEqual} = options;

  if (options.events) {

    function Wrapped(props) {
      const memoEvents = useRef({}).current;
      const eventNames = Array.isArray(events)
        ? events
        : Object.keys(props).filter((k) => /^on[A-Z]/.test(k));
      for (const e of eventNames) {
        memoEvents[e] = memoEvents[e] || wrapEvent(props[e]);
      }
      return <Component {...props} {...memoEvents} />;
    }

    return reactMemo(Wrapped);
  }

  return reactMemo(Component, propsAreEqual);
}

const ws = new WeakSet();

function wrapEvent(func) {
  if (typeof func !== 'function' || ws.has(func)) return func;
  const wrapped = (...params) => func(...params);
  ws.add(wrapped);
  return wrapped;
}
```

IMHO, the essence of the problem is that the event handler should not be regarded as an ordinary callback.
So we need to add a proxy layer to the event as above.

# Drawbacks

- If the event props does not follow the naming convention, it will cause unnecessary wrapping.
- There will be some performance overhead for detecting event props every render.
- In order to be compatible with the original behavior, the default parameter settings may not be reasonable.

# Alternatives

`useCallback`, `useEvent` and some other wrapper hooks put too much mental load on the user.
User may don not know when to use or not. This can easily lead to premature optimization and reverse optimization.


# Adoption strategy

Release it in a minor. 
Use the linter to advise users to remove `useCallback` wrappers for functions starting with on* or handle*.
Write new documentation teaching common patterns.

Warn for `memo` without `events` option.

# How we teach this

In the `memo` documentation, recommend to use this feature.

In the `useCallback` documentation, emphasis callback and event handler are two different concepts.

# Unresolved questions

- How to teach and warn people not use event handler as callback on render time?
- How to add the similar behavior to `PureComponent` and `shouldComponentUpdate`.
