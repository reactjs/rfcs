- Start Date: 2021-09-08
- RFC PR: https://github.com/reactjs/rfcs/pull/205
- React Issue: https://github.com/facebook/react/issues/15176

# Summary

Callback Refs should allow returning a cleanup function which will be called when the ref is removed or changed.

# Basic example

```jsx
function MyComponent(props) {
  const exampleCallbackRef = useCallback((el) => {
    /* effect code */
    return () => { /* cleanup code */ };
  }, [])

  return <button ref={exampleCallbackRef} />;
}
```

# Motivation

Similar to `useEffect`, Callback Refs often create resources that need cleaning up
when an element is unmounted. In such situations, developers often keep a reference to
the element in a mutable ref, and clean it up the next time Callback Ref is 
called with a `null` (See section `Alternatives`). But, as the example above shows, 
it is easier to clean up the effects of a Callback Ref when the element is already in the 
scope of the cleanup function.

The lack of a cleanup mechanism becomes a problem when trying to use the same Callback Ref 
for multiple elements. Consider the following example:

```jsx
function exampleCallbackRef (el) {
  /* effect code */
  return () => { /* cleanup code */ };
}

function MyComponent(props) {
  return <>
    <button ref={exampleCallbackRef} />
    <button ref={exampleCallbackRef} />
    <button ref={exampleCallbackRef} />
  </>;
}
```

With the new feature, it will be possible to know which element is being unmounted in the cleanup code.

# Detailed design

The general design of the feature looks like this:

```jsx
<div ref={node => {
  // Normal ref callback

  return () => {
    // Cleanup function which is called when the ref is removed or changed
  };
}} />
```

The behavior of the cleanup function should be similar to the behavior of the
cleanup function of `useEffect`. Some points to keep in mind are:
- The return value of the Callback Ref should be ignored if it's not a function.
The return value should be called only if it is a function and there should not be
an error when it is `null`, `undefined` or another type that is not a function.
- It should be possible to conditionally return a cleanup function. This means the user
can choose not to have a cleanup function for specific values of a ref.

No changes needs to be done to the other ref types, that is mutable refs and string refs.

## Handling of the last null value

In the current implementation, when a ref is unmounted, the Callback Ref is called 
with `null`. To keep backward compatibility, Callback Refs should still be called 
with `null` when the element unmounts.

The cleanup function should be called when the value of a ref changes from one value to 
another. This means that for the following code:

```jsx
function exampleCallbackRef(el) {
  console.log('Callback ref called for:', el);
  return () => console.log('Cleanup called for:', el);
}

...

<button ref={exampleCallbackRef} />
```

The lifecycle of the Callback Ref should look like:

```
Callback ref called for: <button>
Cleanup called for: <button>
Callback ref called for: null
```

Note that the cleanup function is not called for `null` because it is the last value of this ref.

# Drawbacks

- Although very unlikely, existing codebases may be affected from this change. 
See the section `Adoption strategy` for counter measures.
- This feature will encourage a code pattern where a single Callback Ref can be 
used for multiple nodes. It should be confirmed that this conforms to the React
design principles.

# Alternatives

## Existing workarounds

Users have created custom hooks by using `useRef` and `useCallback`. Basically, 
a mutable ref holds the reference to the cleanup function that was returned 
from the Callback Ref, and that cleanup function is called the next time when the 
Callback Ref is called with a null. 
See implementation of [@huse/effect-ref](https://github.com/ecomfe/react-hooks/blob/master/packages/effect-ref/src/index.ts)
for such an example.

However, none of the alternatives solve the problem introduced by using the same Callback Ref for multiple elements.
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

## Alternative designs

### Wrapping the callback with a symbol

One advanced alternative which eliminates any breaking changes, and which may be an 
overkill, is creating a new hook (`useCallbackRefWithCleanup`). This new hook will 
return an object which uses an internal Symbol (`INTERNAL_SYMBOL_REF_CLEANUP`) to keep 
a reference to cleanup function. If this symbol does not exist in the return value of 
Callback Ref, it will be ignored. The cleanup function will behave the same as originally 
described above.

```js
useCallbackRefWithCleanup((el) => {
  /* effect code */
  return () => { /* cleanup code */ };
});

// The above code will be the same as:

useCallback((el) => {
  /* effect code */
  return { [INTERNAL_SYMBOL_REF_CLEANUP]: () => { /* cleanup code */ } };
});
```

Since `INTERNAL_SYMBOL_REF_CLEANUP` does not exist in the old React versions, there
is no chance that existing codebase will break because of this change.

### Introducing a new ref type

Another alternative is to introduce a new ref type which eliminates confustion 
caused by calling the ref with `null` when the element unmounts.

The main problem with the current implementation of `ref` is that it is not possible to
always know which element was unmounted. So another ref type may be needed. 
This ref type may have the signature (Typescript):

```tsx
interface RegisterRef<T> {
  register: (ref: T) => void;
  unregister?: (ref: T) => void;
}
```

And this can be used like:

```jsx
const onClick = () => console.log('clicked!');

const logClicks = {
  register: (node) => node.addEventListener('click', onClick);
  unregister: (node) => node.removeEventListener('click', onClick);
};

function MyComponent(props) {
  return <button ref={logClicks} />;
}
```

Instead of `register/unregister`, other terminology like `mount/unmount` or `connect/disconnect`
can also be used.

Just like `MutableRef`, which uses `current` property to keep the reference of the element, 
this ref type is also object typed. React should check if the `register` property of the ref is 
a function to disambiguate between `RegisterRef` and `MutableRef`.

Although this solution is quite different than what is discussed in this RFC, it solves the 
same problems. It also eliminated any breaking changes. However, it can be a more significant 
change and require more changes in documentation.

# Adoption strategy

This is a new feature and it is not a breaking change. However, existing projects may 
still be affected if they have been returning a function from a Callback Ref. This would 
be a no-op in previous React versions, but could result in errors in the new version.

A codemod can check if users have been returning a function from Callback Refs and warn them
to manually remove return valeus from Callback Refs before updating React.

# How we teach this

Developers are already familiar with the cleanup feature of `useEffect` hook. It should be 
easy to teach people how to use this feature by explaining it in the same way as 
[how cleanup up of useEffect works](https://reactjs.org/docs/hooks-reference.html#cleaning-up-an-effect).

The phrase "Callback Ref cleanup" already gives an impression of what this feature does. 
The part of documentation that needs to be updated is the section where 
[Callback Refs](https://reactjs.org/docs/refs-and-the-dom.html#callback-refs) 
are explained.

# Unresolved questions

- What should happen when a ref is unmounting? To keep the backward compatibility, the Callback Ref should still be called with `null`.
But, should the cleanup function be called, and if so, when should it be called?
- There are multiple alternative solutions. Which one is more suitable?
- The original proposal will not be a breaking change for the majority of codebases (I would say %99.9). Do we still need alternatives?
