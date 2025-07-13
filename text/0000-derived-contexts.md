- Start Date: 2025-07-12
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Selectors for Context are an in-demand feature, but with a slightly different interface, can be made much more universally useful
([.slice as proposed here is similar](https://github.com/reactjs/rfcs/pull/119#issuecomment-512529871])).

Following from functional programming techniques, this RFC proposes adding:
- a `ReadonlyContext<T>` type which is a supertype of `Context<T>` without `.Provider`

and these methods to Context:
- `.map`, of type: `<T, U>(this: ReadonlyContext<T>, fn: (data: T) => U) => ReadonlyContext<U>`
- `.apply`, of type: `<T, U>(this: ReadonlyContext<T>, fn: ReadonlyContext<(data: T) => U>) => ReadonlyContext<U>` (_I actually recommend one of its alternatives_)

`.map` allows users to refine and select their Contexts, while `.apply` allows users to combine multiple Contexts together. This is covered in detail in Motivation.

Every call, without respect to the identity of the given argument (function / context of functions), returns a distinct Context.
To be usable within renders, existing user-land techniques like WeakMap can be used to stabilize the identity of the returned Context to the identity of the given argument.

# Basic example

```jsx
// User can set either of these for our API
const ViewportReadonly = createContext(null);
const ViewportWritable = createContext(null);

function IntegrateLibrary({ viewport }) {
  return (
    // User can choose which to provide
    <ViewportReadonly.Provider value={viewport}>
      <Library />
    </ViewportReadonly.Provider>
  );
}

// Now ViewportRead will grab either, with a preference for ViewportWritable's
const ViewportRead = ViewportReadonly.apply(ViewportWritable.map((writableViewport) => (readableViewport) => {
  return writableViewport ?? readableViewport;
}));
// or using .with:
const ViewportRead = ViewportWritable.with(ViewportReadonly, (writableViewport, readableViewport) => {
  return writableViewport ?? readableViewport;
});
function Library() {
  // Library will pick up either 
  const viewport = useContext(ViewportRead);
}
```


# Motivation

Motivation for selecting Contexts is already well-established by the popularity of [RFC 119](https://github.com/reactjs/rfcs/pull/119).

Motivation for this approach is that these `.map` and `.apply` functions are drawn from the Functor and Applicative Functor (respectively) from Haskell.
This style of approach is thoroughly treaded ground, being used with great success in Haskell for many years.

To briefly describe the purpose of `.apply` in addition to `.map` (this explanation is no different from the many articles on Functor vs Applicative Functor out in the wild):

- With just `.map` we can refine one Context into any number of others, which can themselves be refined, creating a tree of ever more specific (and less informative) Contexts.

- With `.apply` however, we can merge any number of Contexts into one Context.
Now we can create a DAG, which is so much more expressive than just a tree.

While I believe this interface is more pleasant in both ergonomics, efficiency and theory than wrapping `useContext`,
what I believe is particularly powerful about it is **its ability to change over time**.
If some code depends on a Context, that Context can later be rewritten in terms of other Contexts.
**The consumption is isolated from the production**: the producers can change without necessitating the consumers to read new Contexts.

As an added bonus over wrapping `useContext`, derived Contexts could be used conditionally by `use`.

# Detailed design

Types would be changed like so:

```ts
interface ReadonlyContext<T> {
  Consumer: Consumer<T>;
  displayName?: string | undefined;
}
interface Context<T> extends ReadonlyContext<T> {
  Provider: Provider<T>;
}
```

## Semantics

Contexts from `createContext` now have two prototype methods, `.map` and `.apply` (or `.with`).
The distinct Contexts returned from these methods do not have a `.Provider`.

`.map` has two guarantees, the first that given:
```ts
type T = /* ... */;
type U = /* ... */;
declare function someMapping(data: T): U;
declare function showAsJSX(data: U): ReactNode;
declare const SomeContext: ReadonlyContext<T>;
```

Then, this `SomeComponent`:
```ts
const MappedContext = SomeContext.map(someMapping);
function SomeComponent() {
  const data = useContext(MappedContext);
  return showAsJSX(data);
}
```

will always render the same as this `SomeComponent`:
```ts
function SomeComponent() {
  const data = someMapping(useContext(SomeContext));
  return showAsJSX(data);
}
```

The second is that in this snippet, `Child` will never be rerendered (as least by the `useContext`).
When the map function returns the same as it did for the last push by `Object.is` semantics, the returned Context will not push an update to its subscribers.
```tsx
const CountContext = createContext(0);
const ZeroContext = CountContext.map(() => 0);

function Child() {
  const zero = useContext(ZeroContext);
  return <div>{zero}</div>;
}
const child = <Child />;

function Parent() {
  const [count, setCount] = useState(0);
  return (
    <div onClick={() => setCount(n => n + 1)}>
      <CountContext.Provider value={count}>
        {child}
      </CountContext.Provider>
    </div>
  );
}
```

`.apply` has similar semantics with render equivalence and not pushing updates on the same value.

## Implementation

TODO: How is this implemented?

# Drawbacks

## Surface area

Deriving values from multiple Contexts is very possible with custom hooks wrapping `useContext`.
I believe it is worth the surface area as:

- custom hooks wrapping `useContext` will not be allowed to be called conditionally with `use`.
- anything wanting to preserve `use` semantics could not `useMemo` to avoid expensive recomputation, where `.map` allows that expensive computation to be lifted out of renders.

I also believe the surface area is satisfying small, with no extra hooks or exports.

## ReadonlyContext

Contexts derived from other Contexts cannot have `.Provider`'s.
If derived Contexts had been around from day one of Context, I think distinguishing a `WritableContext` type would be more useful than distinguishing a `ReadonlyContext` (as not many places where the types can't be inferred would need to then provide a Context value).
I would not think a codemod to rewrite current `Context<T>`'s to `WritableContext<T>`'s would be worth it.

However, this only affects TypeScript users, and the vast majority of usages won't have explicit type annotations.

## Implementation complexity

TODO: I do not know how complex this is. 

# Alternatives

## Alternatives of this proposal

Unfortunately Applicative Functors are fairly more awkward in JavaScript than they are in Haskell (Haskell curries by default, where JS doesn't).

The Applicative Functor interface could be made available in some other forms instead:

### Via liftA2

Instead of a type like `.apply`'s:

```ts
<T, U>(this: ReadonlyContext<T>, fn: ReadonlyContext<(data: T) => U>): ReadonlyContext<U>
```

a function called something like `.with` could be exposed:

```ts
<T, U, R>(this: ReadonlyContext<T>, other: ReadonlyContext<U>, combine: (a: T, b: U) => R): ReadonlyContext<R>
```

Note this is the same as `.map`'ing `other` by `combine` into `(data: T) => R`. Similarly `.apply` could be defined in terms of `.with`.

**I recommend this approach**.
I think it would be more natural for JavaScript users, and considering the typical use of combining two Contexts, it only requires creating the output Context,
where `.apply` requires creating the intermediate Context from `.map`'ing `other`.
I didn't want to open the RFC with this, as Applicative Functors are instead usually introduced with `.apply`.

I don't love the name `.with` either.
For reference in Haskell `.with` is called `liftA2`.

### Via sequence

Promises are also valid Applicative Functors. While `.then` does have all the power of `.apply` (and then some), there is a particularly Applicative-y util for Promises: `Promise.all` (_handling of TypeScript tuples omitted for brevity_):

```ts
<T>(promises: Array<Promise<T>>): Promise<Array<T>>
```

This pattern is called `sequence` in Haskell. Exposing a function like:

```ts
<T>(contexts: Array<ReadonlyContext<T>>): ReadonlyContext<Array<T>>
```

would give the same power to users as `.apply`.

However, there are a number of reasons this interface isn't that great:
- combining more than two or three Contexts is not that common, for which the Array is needless overhead
- there is no Context namespace, so it would need to be exposed as a plain function from 'react'. I don't know what'd be a nice name for it.

However, I figured mentioning `Promise` would be useful for precedence.

## Alternatives to this proposal

### Wrapping `useContext`

As already noted, a custom hook wrapping `useContext` can no longer be called conditionally like `use` can.
Additionally, wrapping `useContext` will always be subscribed to any change, where derived Contexts will stop repeat pushes.

However, a custom hook wrapper has a lot of power as the code using it evolves over time.
It can be made to accept arguments, or at some point switch to actively procuring the data.

### [useContextSelector](https://github.com/reactjs/rfcs/pull/119)

At an API level, this proposal competes with `useContextSelector` the most.
In those terms, I think derived Contexts are the better first option:
- `useContextSelector` can be defined in terms of derived Contexts, both in user-land and in React itself
- Derived contexts greatly increase the expressiveness of Context as a tool of composition, not just for state management
- No questions about identity ("do I need to useMemo the selector?"): users expect `.map` to change identity

However, at the performance level, derived Contexts might be able to be made performant enough to satisfy `useContextSelector`'s use cases.
Then the more universal applicability of derived Contexts could justify adding (it as) a solution for those cases.

# Adoption strategy

No existing APIs are modified so it is opt-in only.

A codemod for TypeScript users _could_ be made to convert existing explicit type annotations of `Context<T>`'s to `WritableContext<T>`'s.

# How we teach this

These methods could share one docs page, and beyond that would be out of the user's way.
For the users who stumble upon it while poking around:

- `.map` is already a familiar name from Array, and
- they are completely described by their types (not preserving identity is their only non-pure semantic, and that has strong precedence from Array `.map`).

Finding an intuitive name for `.with` would be the trickiest part.

# Unresolved questions

- `Context<T>` + `ReadonlyContext<T>` vs `WritableContext<T>` + `Context<T>`
- Implementation complexity and performance of derivation
- Could default displayName of derived Contexts from input Context displayName's and the function names
