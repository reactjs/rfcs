- Start Date: 2021-09-08
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Callback Refs should allow returning a cleanup function which will be called when the ref is removed or changed.

# Basic example

```jsx
function MyComponent(props) {
  const logClicks = useCallback((node) => {
    if (!node) return;

    const onClick = () => console.log('clicked!');

    node.addEventListener('click', onClick);

    return () => {
      node.removeEventListener('click', onClick);
    };
  }, [])

  return <button ref={logClicks} />;
}
```

# Motivation

Similar to `useEffect`, Callback Refs often create resources that need cleaning up
when an element is unmounted. In such situations, developers often keep a reference to
the element in a mutable ref, and clean it up the next time Callback Ref is 
called with a `null` (See section `Alternatives`). As in the basic example above, 
it is easy to clean up the effects of a Callback Ref  when the element is already in the 
scope of the cleanup function.

This feature also opens the way for two useful code patterns:

### Standalone Callback Refs

In the example above, the Callback Ref can be moved out of the function component. 
Because the callback does not need to keep a ref anymore.

```jsx
function logClicks (node) {
  if (!node) return;

  const onClick = () => console.log('clicked!');

  node.addEventListener('click', onClick);

  return () => {
    node.removeEventListener('click', onClick);
  };
}

function MyComponent(props) {
  return <button ref={logClicks} />;
}
```

### Register pattern

The same Callback Ref can be used for multiple elements.

```jsx
function MyComponent(props) {
  return <>
    <button ref={logClicks} />
    <button ref={logClicks} />
    <button ref={logClicks} />
  </>;
}
```

For example, we can write a Callback Ref that adds all elements to an array, 
and remove them from the array when they are removed from DOM:

```jsx
function Test() {
  const listRef = useRef([]);

  const register = useCallback((el) => {
    if (!el) return;

    listRef.current.push(el);
    
    return () => {
      const index = listRef.current.indexOf(el);
      if (index >= 0) listRef.current.splice(index, 1);
    };
  }, []);

  return <>
    {[0,1,2,3,4,5].map((x,i) => (
        <span key={i} ref={register} />
    )}
  </>;
}
```

See the section `Alternatives` for how `react-hook-form` uses the `register` pattern.

# Detailed design

The general design of the feature looks like this:

```jsx
<div ref={node => {
  // Normal ref callback

  return () => {
    // Cleanup function which is called when the ref is removed or changed
    // This should not be called when `node` is null
  };
}} />
```

To keep backward compatibility, Callback Refs should still be called with `null` 
when the element unmounts. But in this case, the value returned from Callback Ref 
should be ignored, i.e. the cleanup function should never be called.

No changes needs to be done to the other ref types, mutable refs and string refs.

# Drawbacks

- Existing codebases may be affected from this change. See the section `Adoption strategy` 
for counter measures.
- This feature will encourage a code pattern where a single Callback Ref can be 
used for multiple nodes. It should be confirmed that this conforms to the React
design principles.

# Alternatives

Users have created custom hooks by using `useRef` and `useCallback`. Basically, 
a mutable ref holds the reference to the cleanup function that was returned 
from the Callback Ref, and that cleanup function is called the next time when the 
Callback Ref is called with a null. 
See implementation of [@huse/effect-ref](https://github.com/ecomfe/react-hooks/blob/master/packages/effect-ref/src/index.ts)
for such an example.

However, none of the alternatives solve the problem introduced by `register` pattern. 
Although not as a drop-in replacement, workarounds exist. One of the libraries that use
this pattern is `react-hook-form`, and it solves this issue by making `register` a function
that returns the actual Callback Ref when called with a unique ID:

```
<input ref={register("email")} />
<input ref={register("password")} />
```

In the previous versions of `react-hook-form`, the API used to look like:

```
<input name="email" ref={register} />
<input name="password" ref={register} />
```

This API was abandoned, presumably because how hard it is to maintain a registry of
elements without a cleanup function. Note that we are not discussing which API 
looks better. But having the cleanup functionality will be useful for other library 
developers as well. Also, unlike this example, it is not always convenient to come 
up with unique names for each element.

# Adoption strategy

This is a new feature and it is not a breaking change. However, existing projects may 
still be affected if they have been returning a value from a Callback Ref. This would 
be a no-op in previous React versions, but could result in errors in the new version.

To be on the safe side, adoption of this feature can be done in two steps:

- Modify the React source code so that returning a value from a Callback Ref is 
forbidden. We can also add Flow and Typescript types to statically catch such situations.
- Implement this feature, preferably in a major release.

Alternatively, a codemod can check if users have been returning anything from Callback Refs and warn them before updating React.

One advanced alternative which eliminates any breaking changes, and which may be an 
overkill, is creating a new hook. This new hook will return an object which uses 
an internal Symbol to keep a reference to cleanup function. If this symbol does not
exist in the return value of Callback Ref, it will keep being a no-op.

# How we teach this

Developers are already familiar with the cleanup feature of `useEffect` hook. It should be 
easy to teach people how to use this feature by explaining it in the same way as 
[how cleanup up of useEffect works](https://reactjs.org/docs/hooks-reference.html#cleaning-up-an-effect).

The phrase "Callback Ref cleanup" already gives an impression of what this feature does. 
The part of documentation that needs to be updated is the section where 
[Callback Refs](https://reactjs.org/docs/refs-and-the-dom.html#callback-refs) 
are explained.

# Unresolved questions

- What should happen when Callback Refs return something other than a function?
- Should there be a change to how Callback Refs are called with `null`?
