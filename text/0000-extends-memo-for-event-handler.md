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
import {memo as reactMemo, forwardRef, useRef} from 'react';

export default function memo(WrappedComponent, options) {
  if (!options || typeof options === 'function') {
    return reactMemo(WrappedComponent, options);
  }

  let Wrapped = WrappedComponent;
  const {events} = options;
  if (events) {
    // events is an array or true
    Wrapped = forwardRef((props, ref) => {
      const memoEventsRef = useRef({});
      const eventNames = Array.isArray(events)
        ? events
        : Object.keys(props).filter((k) => /^on[A-Z]/.test(k));

      const memoEvents = eventNames.reduce(
        (me, n) => ({
          ...me,
          [n]: fixEventHandler(memoEventsRef.current[e], props[e])
        }),
        {}
      );
      memoEventsRef.current = memoEvents;
      return <WrappedComponent ref={ref} {...props} {...memoEvents} />;
    });

    Wrapped.displayName = `FixedEvent(${getDisplayName(WrappedComponent)})`;
  }

  return reactMemo(Wrapped, options.propsAreEqual);
}

const wm = new WeakMap();

function fixEventHandler(fixed, handler) {
  if (typeof handler !== 'function' || wm.has(handler)) return handler;
  if (!fixed) {
    fixed = (...params) => {
      const h = wm.get(fixed);
      return h(...params);
    };
  }

  wm.set(fixed, handler);
  return fixed;
}

function getDisplayName(WrappedComponent) {
  return WrappedComponent.displayName || WrappedComponent.name || 'Component';
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
