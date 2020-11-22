- Start Date: 2020-11-22
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Introduce an alternative to `createContext` that accepts an explicit unique
identifier instead of allocating an object with a new identity. This would
enable explicit sharing of context across module loading boundaries, for
example when doing server-side rendering (SSR) with code splitting and multiple
entrypoints.

By analogy, a `contextFor(uniqueKey, defaultValue)` function would behave like
[Symbol.for][1] in that subsequent calls with the same argument return the same
result. Unlike JavaScript symbols, however, this result should be useful for
distributed execution environments. In particular, a provider and consumer from
independent compilation units should be compatible if they were constructed
with the same unique identity.


# Basic example

```javascript
// The first argument can be any string we can arrange to be unique across
// packages. For example, here we choose a URL from a domain we control.
const MyContext = React.contextFor('example.com/my-context', defaultValue);
```


# Motivation

Currently, `React.createContext` allocates a new object and uses the identity
of that object to uniquely identify context providers from the point of context
consumption. This works great within a single execution environment, such as
that of a web browser. However, when server-side rendering (SSR) is involved,
things can get much more complex. For example, Next.js does some truly hacky
manipulation of the Webpack module instantiation configuration to address these
kind of "singleton" usages of static state. here are some associated issues:

- https://github.com/vercel/next.js/issues/18917
- https://github.com/vercel/next.js/blob/f637c8a2ccdb9a3ac695a0c303bd34f14655633d/packages/next/build/webpack/plugins/nextjs-ssr-module-cache.ts

This is a well known issue with static state and distributed execution
environments. As is the fix with any implicit identity in a distributed
environment, introducing an explicit identifier can untangle the problem.

# Detailed design

```javascript
const myContextKey = 'example.com/my-context';

// Besides the explicit context key, this behaves like `createContext`.
const MyContext = React.contextFor(myContextKey, defaultValue);

// Calling `contextFor` again returns an object that is `contextEqual`.
const MyContext2 = React.contextFor(myContextKey, defaultValue);
assert(React.contextEqual(MyContext, MyContext2));

// These two `Context` instances are not `===` but are compatible:
const Outer = ({ children }) => (
  <MyContext.Provider value={123}>{children}</MyContext.Provider>
);
const Inner = ({ children }) => {
  const value = useContext(MyContext2); // Note the "2".
  assert(value === 123);
  return <div>{value}</div>;
);
const Main = () => <Outer><Inner/></Outer>;
```

# Drawbacks

- A cursory examination of the React implementation suggests that the object
  identity of the Context object is used to implement fast context resolution.
  Introducing an explicit unique ID would necessitate a slower equality test and
  introduce an extra indirection to dereference the current context value.
- Module instantiation caches for Webpack may still be necessary for projects
  that use `createContext` instead of `contextFor`, or will have to be updated
  to simplify/improve SSR support.
- Library designers will need to be taught when and how to select a unique ID.

# Alternatives

- React could provide some lower-level hooks for Webpack, other bundlers, or
  other module loading contexts to better share state.

# Adoption strategy

Breaking change for any library that uses `===` directly or indirectly on
Context objects that they did not create themselves. This is likely to be quite
rare, but almost certainly exists in some framework somewhere.

# How we teach this

Progressive disclosure: Recommend `createContext` unless you know you have a
need for `contextFor`. Document `createContext` as being implemented in
terms of `contextFor`:

```javascript
const createContext = (defaultValue) => contextFor(Symbol(), defaultValue)
```


[1]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol/for
