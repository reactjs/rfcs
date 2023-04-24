- Start Date: 2023-01-31
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Introduce a new built-in API for creating hooks which will be automatically
hoisted. This would address the desire for state sharing across components in
a more composable, explicit and flexible way.



# Basic example

Sharing the count state between two Counters within the same scope

```jsx
import { createScope, createStore, useStore } from "react"

const CounterScope = createScope ()
CounterScope.displayName = "CounterScope"

const countStore = createStore (() => {
  const [ count, setCount ] = useState (0)
  return { count, increment }
}, [ CounterScope ])

function Counter () {
  const { count, increment } = useStore (countStore)
  // ...
}

function App () {
  return <>
    <CounterScope>
      <Counter />
      <Counter />
    </CounterScope>
  </>
}
```



# Motivation

The Provider pattern is our de-facto solution to hoisted state but it has a 
fundamental limitation: we cannot add Providers during the runtime without side 
effects caused by re-mounting the tree.

Situations where Providers are insufficient:
- Lazily loading hoisted state like for Micro-Frontends or plugins.
- Performance when hoisting state of list items.

This approach has some ergonomic benefits as well:
- Context can be used for more primitive/intrinsic values.
  - This is a more natural role for Context.
- Business logic can be explicitly imported by it's consumer.
- More compatible with Suspense.
- No duplication necessary.
- Avoiding re-renders from derived state is trivial.
  - Pattern is resilient to duplication.

I think this could potentially become the industry standard over third-party 
libraries.



# Detailed design

> This is the bulk of the RFC. Explain the design in enough detail for somebody
> familiar with React to understand, and for somebody familiar with the
> implementation to implement. This should get into specifics and corner-cases,
> and include examples of how the feature is used. Any new terminology should be
> defined here.

TODO

## Global Scoping

To ensure a store is globally scoped an empty scopes list is used.

### Example

```tsx
const canScrollStore = createStore (() => {
  const [ canScroll, setCanScroll ] = useState (false)
  
  useLayoutEffect (() => {
    // enable/disable scrolling
  }, [ canScroll ])
  
  return { canScroll, setCanScroll }
}, [])

function Modal () {
  const { canScroll, setCanScroll } = useStore (canScrollStore)
  
  useEffect (() => {
    setCanScroll (false)
  }, [])
  
  // ...
}
```



## Local Scoping

Stores without a scopes list parameter are "local", they are scoped to whatever
uses them. This is helpful for two use cases:

1. Conditional Hook Calls
2. Passing and using a "custom hook"

### Example

`NavbarContent` can be just the navbar or it can have a menu of data inside of it.
That menu and whether or not it should be displayed can be changed from page to 
page and that logic can be passed through context using a local store.

```tsx
const menuStore = createStore (() => {
  // ...
})

const MenuStoreContext = createContext (menuStore)

function Navbar ({ simple }) {
  const menuStore = useContext (MenuStoreContext)
  const menu = simple ? undefined : use (menuStore)
  return <NavbarContent menu={menu} />
}
```



## Use Case: Store Family

One of the most valuable attributes of `createStore` is that any number of
stores can be created which means each item in a dynamically sized list could
have a store. This is based on Recoil's `atomFamily` utility.

A utility like that doesn't need to be a part of React but here's what it might
look like:

```jsx
function createStoreFamily (hook, deps) {
  return _.memoize ((key) => {
    return createStore (() => hook (key), deps)
  })
}
```

### Pitfall: Memory Leak

This pattern is prone to memory leaks, however, that's not really a React problem
unless it's decided to that `createStoreFamily` is important enough to be included
in the core library (which I think is perfectly reasonable).

### Example

```jsx
const AccordionScope = createScope ()

const openIdStore = createStore (() => {
  const [ openId, setOpenId ] = useState (null)
  return { openId, setOpenId }
}, [ AccordionScope ])

const openStoreBy = createStoreFamily ((id) => {
  const { openId, setOpenId } = useStore (openIdStore)
  const open = openId === id
  const toggleOpen = useEvent (() => setOpenId (open ? null : id))
  return { open, toggleOpen }
}, [ AccordionScope ])

function AccordionItem ({ id, children }) {
  const { open, toggleOpen } = use (openStoreBy (id))
  return (
    <Expandable open={open} onToggle={toggleOpen}>
      {children}
    </Expandable>
  )
}
```



## Errors & Suspense

Thrown errors and promises would be expected to route through the components
which use the store. This is really the only viable approach otherwise
there would be huge potential unintended side effects.

This is how throwing works in libraries like Jotai and Recoil.

### Example

Layout should render the same with or without the "middleman" stores here. The
children are rendered immediately, the Footer and Header both trigger their
Suspense boundaries and then Footer and FallbackHeader are rendered.

```tsx
function Layout ({ children }) {
  return <>
    <ErrorBoundary fallback={<HeaderFallback />}>
      <Suspense fallback={<HeaderLoader />}>
        <Header />
      </Suspense>
    </ErrorBoundary>
    { children }
  </>
}

// Header

const fetchHeaderData = cache (async () => {
  throw new Error ("No Header Data")
})

const headerDataStore = createStore (() => {
  return use (fetchHeaderData())
}, [])

function Header () {
  const data = useStore (headerDataStore)
  // ...
}
```



## Best Practices

There are a couple keys to using Scope effectively that I think need to be called out.

1. Context and Scope are often two sides of the same concept, they should be co-located.
2. If a store calls another store, it should depend on the same scopes as that store. 
To make that easier, we should allow for stores to be passed as if they were scopes.

### Example

```jsx
// ./product.js

import { createContext, createStore, createScope } from "react"

const ProductScope = createScope ()
const ProductIdContext = createContext ()

export const ProductProvider = ({ children, id }) => (
  <ProductIdContext.Provider key={id} value={id}>
    <ProductScope>
      {children}
    </ProductScope>
  </ProductIdContext.Provider>
)

export const productIdStore = createStore (() => {
  return useContext (ProductIdContext)
})

// ./selected-color.js

const selectedColorStore = createStore (() => {
  const id = useStore (productIdStore)
  
  const [ defaultColor ] = use (fetchProductColors (id))
  const [ selectedColor, setSelectedColor ] = useState (defaultColor)
  
  return { selectedColor, setSelectedColor }
}, [ productIdStore ])
```



# Drawbacks

- The implementation would be non-trivial.
- Can be used in a way that behavior depends on memoization.
- Educating developers on an additional concept and API.
- This is yet another state management solution.

### Difficult Implementation

From looking at the source code, it looks like it would require a good amount
of refactoring of the internals to get this to work. The big issue is the
initial execution

### Global State

It's been previously mentioned that global state in general kind of violates the
React team's model for composability. The solution I've proposed does allow for 
global state but we could do things differently if needed.



# Alternatives

The status quo is viable for React, this is more of an opportunity. The issues
addressed by `createStore` are some of the most common in the React community and 
a good number of the RFCs submitted are related to state management in one way or 
another.



## Auto-Scoping Hoist API

There is an alternate API that is more similar to the original version of this RFC.
This variant would come with some tradeoffs but is arguably better as it uses Context
to automatically determine the "scope" to hoist the "store" to.

### Example

This is equivalent to the "Best Practices" example.

```jsx
import { context, hoist } from "react"

// ./product.js

export const [ ProductIdProvider, useProductId ] = context ()

// ./selected-color.js

const useColorState = hoist (() => {
  const id = useProductId ()
  
  const [ defaultColor ] = use (fetchProductColors (id))
  const [ selectedColor, setSelectedColor ] = useState (defaultColor)
  
  useEffect (() => {
    setSelectedColor (defaultColor)
  }, [ id ])
  
  return { selectedColor, setSelectedColor }
})
```

### How does `hoist` relate to `createStore`?

Hoist is basically a wrapper around `createStoreFamily`. In both variations, there is a
`createStore`, but in this variation `createStore` is an internal because it's not
ever meant to be referenced directly.

In the default version of this proposal, `createStore` has deps so `hoist` would also need
a `deps` argument. But in this alternative proposal, no `deps` is needed.

```js
function createStoreFamily (hook) {
  return _.memoize ((key) => {
    return createStore (() => hook (key))
  })
}

function hoist (hook) {
  const family = createStoreFamily (hook)
  return (...args) => useStore (family (...args))
}
```

### Drawback: A Different Context API

This would require a Context API that is either new or is a breaking change. The current
`useContext` implementation allows it's argument to change but that would break this
"auto-scoping" proposal. The context must stay the same.

I assume this is a dealbreaker but maybe with RSC and it's conflicts with Context, I'm
wrong and a new Context API is on the table.

### Drawback: Incompatibility with conditional `use`

There may be some wiggle room but generally, this "auto-scoping" variant isn't compatible
with conditional execution because - like the drawback above - the scopes need to be 
consistent.



## Partial Alternatives

These are options we have for addressing the same issue but in a less complete manner.

### Provider API

If we had an API for making Context Providers that could be added lazily 
without unmounting the tree underneath that would unlock addressing some of the
problems that `createStore` is addressing but it would not be close to equivalent.

### Atomic Library

The only alternative to a built-in API is a Recoil-like library. I have created
one and will hopefully be able to open source it soon. Still, that isn't quite
equal to having that full integration of a built-in API



# Adoption strategy (TODO)

I think this is a similar kind of change to when hooks were added. 
It's a significant addition that would effect how React developers write their code
but it's not a breaking change to the library and it can be adopted incrementally.

It's relationship with Context could make adoption more complicated though. It
seems like a non-issue to me right now but I could be wrong.



# How we teach this

### Terminology (TODO)

### Presentation

The first 8 minutes on this video about Recoil do a fantastic job of talking
about this problem: https://www.youtube.com/watch?v=_ISAA_Jt9kI&t=3s. We then
slide in stores instead of atoms and then add scopes for non-global state.

We can then present it again from the "bottom-up" to show how this is another
form of composition in React as we're now explicitly importing from the same
area as the business logic exists.

### Documentation (TODO)

This would involve changes to the React documentation, mostly additions.
I think this would become a major part of the suggested approach to React
development. I assume the Provider pattern isn't more prominently featured
because it's a pattern as opposed to an API and not a particularly popular one.



# Unresolved questions (TODO)

- Does this conflict with lesser-known React APIs current or future?
- What would need to change internally to support this?
- Is it okay if it's always memoized?
- What would be needed for this to fit the React team's vision? 
(see: https://github.com/reactjs/rfcs/pull/130#issuecomment-901475687)
