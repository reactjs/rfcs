- Start Date: 2021-08-25
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Add a status argument to the useEffect/useLayoutEffect callback.

# Basic example

```js
useEffect(status => {
  asyncOp(arg).then(res => {
    !status.aborted && setState(res);
  });
}, [arg]);
```

# Motivation

When ever we use an effect with a state update inside an async callback (or after an `await`) we probably want to check that we are not trying to update an unmounted component or worse, we are in a race condition and updating the state with the wrong data.

This is fairly simple to achieve using a boolean and a closure:

```js
useEffect(() => {
  let current = true;
  asyncOp(arg).then(res => {
    current && setState(res);
  });

  // cleanup
  return () => current = false
}, [arg]);
```

But doing that in each effect, every time we are dealing with a state update in an async operation can become tedious.
It would be nice if effects would provide this info so users won't have to create booleans and return a function that flip them every time.

Another possible issue, is the fact that some developers doesn't think about cleanups this way, cleaning up is usually done to explicit APIs like subscribers or some well known browser APIs such as `setTimeout` and `setInterval`, but some (or most) developers don't think about effect cycles and async updates while writing the code, until they hit a bug. Explicitly passing a status to effects callbacks can help developers think and "remember" that their async operation and updates should properly handled when there is a change.

# Detailed design

- The effect callback can accept an object parameter.
- useEffect/useLayoutEffect will pass an object with a boolean property `aborted` (not sure about this name at all) as `false`.
- On cleanup, useEffect/useLayoutEffect will mutate the passed object and will set `aborted` to `true`.

# Drawbacks

- I'm not sure what is the implementation cost, both in term of code size and complexity.
- This [can be implemented in user-land](#Alternatives) (but can get either tedious or "ugly" API).
- Might be a small extra "thing" to teach developers.
- Might be considered "magic" or "black box" by some developers.

# Alternatives

There might be a "user-land" solution by wrapping `useEffect`/`useLayoutEffect`:

```js
function useStatusEffect(effect, dependencies) {
  const status = {}; // mutable status object
  useEffect(() => {
    status.aborted = false;
    // pass the mutuable object to the effect callback
    // store the returned value for cleanup
    const cleanUpFn = effect(status);
    return () => {
      // mutate the object to signal the consumer this effect is cleaning up
      status.aborted = true;
      if (typeof cleanUpFn === "function") {
        // run the cleanup function
        cleanUpFn();
      }
    };
    // not the best way to pass dependencies, but what other options do we have
  }, dependencies);
}
```

And the usage:

```js
useStatusEffect(status => {
  asyncOp(arg).then(res => {
    !status.aborted && setState(res);
  });
}, [arg]);
```

# Unresolved questions

Not sure about the name `aborted`, it is too related to specific use cases. Maybe something like `isCleaned` or something similar.