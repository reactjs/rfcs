- Start Date: 2022-10-21
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

This RFC addresses how the `useCallback` hook is used and how it helps in
inconsistency and encourages memory leaks.

`useCallback` returns a function reference governed by the attached dependencies,
 this function stays alive even if dependencies change.

`useCallback`'s function often calls a state setter and close over some other
variables, then the returned function is usually sent as a callback in an 
asynchronous flow.

Due to the unpredictable nature of async js (we don't know for sure when it will
call our function), the callback may be called after dependencies change and 
obtaining a new function reference, which will lead the component to an
incorrect state. Most of "wrong shown data" errors I debugged had this exact
issue.

# Solution

The proposed solution will be just invalidating the previous function whenever
we change the dependencies and give a new reference. While adding a warning
in development mode to the developer may just unsubscribe to abort its async flow.

One approach I can think of to implement this is something like this:

```typescript

function updateCallback<T>(callback: T, deps: Array<mixed> | void | null): T {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const prevState = hook.memoizedState;
  if (prevState !== null) {
    if (nextDeps !== null) {
      const prevDeps: Array<mixed> | null = prevState[1];
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        return prevState[0];
      }
    }
  }

  const wrappedCallback = realCallback.bind(null, hook, callback, nextDeps);
  hook.memoizedState = [wrappedCallback, nextDeps];

  return wrappedCallback;
}

function realCallback<T>(
  hook: Hook,
  callback: T,
  currentDeps: void | null | undefined | any[]
) {
  if (areHookInputsEqual(currentDeps, hook.memoizedState[1])) {
    return callback.apply(null, arguments);
  } else {
    if (__DEV__) {
      warnAboutStaleUseCallbackCall();
    }
    return undefined;
  }
}

function warnAboutStaleUseCallbackCall() {
  console.error("A stale useCallback's function was called.")
}

```

The drawbacks for this solution is that it does a full traversal of the
dependencies everytime it is invoked, it can be optimized by altering a boolean
variable in the scope of `updateCallback` and will also require creating*
the real callback there.

# Final words

I understand that this _may_ break some already broken software, but I cannot
think of a use case which this invalidation is considered harmful.

I searched over the old issues but did not find any discussion about this.

Best regards.
