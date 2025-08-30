- Start Date: 2025-08-30
- RFC PR: 
- React Issue: 

# Summary

Allow React Context to accept non-`children` props as the provided value. Keep `value` for backward compatibility, but if no `value` is passed, all other props (except `children`) become the context value.

# Basic example

```jsx
<MyContext foo={foo} bar={bar}>
  <App />
</MyContext>
```

Consumer:

```jsx
const { foo, bar } = useContext(MyContext);
```

# Motivation

Currently, React Context requires all values to be wrapped in a single `value` prop, which is inconsistent with how props are normally passed in React.  
This adds boilerplate and can be confusing for beginners. Allowing multiple props to become the context value simplifies the API, reduces wrapping, and aligns Context with normal React patterns.

# Detailed design

- If `value` is present → it is used as the context value (backward-compatible).  
- Otherwise → context value is an object composed of all non-`children` props.  
- `children` is excluded from the context value.  
- Reserved props like `key` and `ref` are ignored as usual.

Example migration:

**Before (legacy pattern):**
```jsx
<MyContext.Provider value={{ theme, locale }}>
  <App />
</MyContext.Provider>
```

**React 19:**
```jsx
<MyContext value={{ theme, locale }}>
  <App />
</MyContext>
```

**Proposed:**
```jsx
<MyContext theme={theme} locale={locale}>
  <App />
</MyContext>
```

# Drawbacks

- Slightly more complex implementation of the Context provider.  
- Potential confusion if `value` and other props are passed together.  
- Need to update documentation and teaching materials.  

# Alternatives

- Keep the current `value={{…}}` pattern.  
- Provide userland wrapper components to wrap multiple props into `value` (less consistent, more scattered).  

# Adoption strategy

- Old code using `value` continues to work.  
- New code can adopt the prop-based pattern gradually.  
- No breaking changes.  
- No codemod is required, but could be optional for large migrations.  

# How we teach this

- Explain that Context now works like normal props.  
- Emphasize that `children` remains special and is not included.  
- Tutorials can now show `<MyContext foo={x} bar={y}>` without wrapping in `value`.  
- Documentation updated to show both legacy (`value`) and new prop-based usage.  

# Unresolved questions

- Should React warn if both `value` and other props are passed?  
- How should TypeScript inference handle the context type when `value` is absent?
