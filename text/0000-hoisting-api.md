- Start Date: 2023-01-31
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Introduce a new built-in API for creating hooks which will be automatically hoisted.
This would address the desire for simplified state sharing in a more composable and flexible manner.

# Basic example

Here is a contrived example with an item in a list that can be expanded.
For a more real world use case see the end of the RFC.

```tsx
import { IdContext, useExpandedState } from "./state"

function ExpandedContent () {
  const [ expanded ] = useExpandedState()
  if (!expanded) return null
  // ...
}

function ExpandButton () {
  const [ expanded, setExpanded ] = useExpandedState()
  // ...
}

function Item () {
  return <>
    <ExpandButton />
    <Content />
    <ExpandedContent />
  </>
}

export function List ({ ids }) {
  return <>
    {ids.map (id => (
      <IdContext.Provider key={id} value={id}>
        <Item />
      </IdContext.Provider>
    ))}
  </>
}
```

## Hoisted to Item-Level

With this version, the expanded state is shared just between the descendents of
the `IdContext.Provider`. If that provider were to unmount all of the expanded
states would also be unmounted.

```tsx
export const IdContext = createContext ("")

const useExpandedState = hoist (() => {
  const id = useContext (IdContext)
  useAssertConstant (id)
  return useState (false)
})
```

## Global State

With this version, the expanded state would be sharable across all components
and would persist for the length of the session.

```tsx
const useExpandedStateById = hoist ((id: string) => {
  useAssertConstant (id) // never
  return useState (false)
})

export const IdContext = createContext ("")

export const useExpandedState = () => {
  const id = useContext (TodoIdContext)
  return useExpandedStateById (id)
}
```

# Motivation

The Provider pattern is our de-facto solution to hoisted state but it has a fundamental limitation:
we cannot add Providers during the runtime without side effects caused by re-mounting the tree.

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

I think this could potentially become the industry standard over third-party libraries.

## Backstory

I was trying to figure out how to share state between components that are distributed and loaded dynamically 
using Module Federation, which the Provider pattern was highly incompatible with. 
I started with Recoil which worked well for global state but I also needed state that was hoisted but not global.

I ended up creating a new atomic state library one which could have state that is contextual instead of global.
After iterating on the API for a while, I realized that I was approaching hooks and 
that this would be better integrated with React given it's reliance on Context.

# Detailed design (TODO)

This is the bulk of the RFC. Explain the design in enough detail for somebody
familiar with React to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

---

Please refer to the Full Example section at the end of the RFC.

# Drawbacks

- The implementation would be non-trivial.
- Arguably, a retcon of the Context API.
- Can be used in a way that behavior depends on memoization.
- Educating developers on an additional concept and API.
- Somewhat overlaps with what already exists.
- This is yet another state management solution.
- Unsure of any conflicts their might be with lesser known features.

## Global State

While `hoist` (as I've described it) could be used like `useGlobalState` (which
has previously been rejected) we could alter the API to prevent that. I'm not
convinced that would be desirable but it is an option.

# Alternatives

What other designs have been considered? What is the impact of not doing this?

---

The status quo is viable for React, this is more of an opportunity. The issues
addressed by `hoist` are some of the most prominent and 

## Similar API

There are a number of ways to implement a hoisting API that works in a similar
way but with different syntax. `hoist` as I've described here is arguably, a
bit too "auto-magical" and that could be confusing.

## Provider API

If we had an API for making Context Providers that could be added lazily 
without unmounting the tree underneath that would unlock addressing some of the
problems that `hoist` is addressing but it would not be close to equivalent.

## Atomic Library

The only alternative to a built-in API is a Recoil-like library. I have created
one and will hopefully be able to open source it soon. Still, that isn't quite
equal to having that full hooks and Context integration of a built-in API.

# Adoption strategy (TODO)

If we implement this proposal, how will existing React developers adopt it? Is
this a breaking change? Can we write a codemod? Should we coordinate with
other projects or libraries?

---

I think this is a similar kind of change to when hooks were added. 
It's a significant addition that would effect how React developers write their code
but it's not a breaking change to the library and it can be adopted incrementally.

# How we teach this

What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing React patterns?

Would the acceptance of this proposal mean the React documentation must be
re-organized or altered? Does it change how React is taught to new developers
at any level?

How should this feature be taught to existing React developers?

---

## Terminology (TODO)



## Presentation (TODO)

The first 8 minutes on this video about Recoil do a fantastic job of talking about this problem: 
https://www.youtube.com/watch?v=_ISAA_Jt9kI&t=3s. 
We present the (prop-less) Provider pattern and it's limitations.
Then we "extract" the Provider from the tree which would be just like a "store".
Then we present it again from the "bottom-up" 
to show how this we can think about components using these stores as composition.
I would adapt that talk to include the big differences from Recoil to the proposed API: hooks and context.

## Documentation (TODO)

This would involve changes to the React documentation, mostly additions.
I think this would become a major part of the suggested approach to React development.
I assume the Provider pattern isn't more prominently featured because it's a
pattern as opposed to an API and not a particularly popular one.
I expect this being an API makes it much more suitable as a best practice.

# Unresolved questions (TODO)

- Does this conflict with lesser-known React APIs current or future?
- What would need to change internally to support this?
- Is it okay if it's always memoized?
- What would be needed for this to fit the React team's vision? 
(see: https://github.com/reactjs/rfcs/pull/130#issuecomment-901475687)





# Full Example

I've added this section with the real world example that led me to this idea.



## Use Case

Here's the features, this will be true for every example. The code will be the
same unless specified otherwise.

### ./provider
```tsx
export const ColorContext = createContext ("")
export const ProductIdContext = createContext ("")

// TODO: add any wrappers as necessary
export const ProductProvider = ({ children, id }) => (
  <ProductIdContext.Provider value={id}>
    {children}
  </ProductIdContext>
)
```

### ./label

Exogenous to the example.
`ProductLabel` displays basic product information.

### ./data

Exogenous to the example.
Hooks which use ProductIdContext and return data from the server.

### ./color/state.tsx
```tsx
export const useSelectedColorState = () => {
  // TODO
  return { selectedColor, setSelectedColor }
}
```

### ./color/swatch.tsx
```tsx
const useColorSelected = () => {
  const color = useContext (ColorContext)
  const { selectedColor } = useSelectedColorState ()
  return color === selectedColor
}

// doesn't use ColorContext for example purposes
const useSelectColor = (color: string) => {
  const { setSelectedColor } = useSelectedColorState ()
  return useCallback (() => {
    setSelectedColor (color)
  }, [ color, setSelectedColor ])
}

export function Swatch ({ color }) {
  const selected = useColorSelected ()
  const onClick = useSelectColorId (colorId)
  // ...
}
```

### ./color/components.tsx
```tsx
import { useProductColors, useProductColorImageUrl } from "../data"

export function ColorField () {
  const colors = useProductColors ()
  return (
    <div>
      {colors.map (color => (
        <ColorButton key={color} color={color} />
      ))}
    </div>
  )
}

export function ProductImage () {
  const { selectedColor } = useSelectedColorState ()
  const src = useProductColorImageUrl (selectedColor)
  return <img src={src} />
}
```

### ./card.tsx
```tsx
import { ProductLabel } from "./label"
import { ColorField, ProductImage } from "./color/components"

export const ProductCard = () => (
  <div>
    <ProductImage />
    <ColorField />
    <ProductLabel />
  </div>
)
```

### ./page.tsx
```tsx
import { ProductLabel } from "./label"
import { ColorField, ProductImage } from "./color/components"
import { Card } from "./card"
import { useRecommendations } from "./data"

const Recommendations = () => {
  const ids = useRecommendations ()
  const cards = ids.map (id => (
    <ProductProvider key={id} id={id}>
      <ProductCard />
    </ProductProvider>
  ))
  return <>{cards}</>
}

const ProductPage = ({ id }) => (
  <ProductProvider id={id}>
    <article>
      <section>
        <ProductImage />
      </section>
      <section>
        <ProductLabel />
        <ColorField />
      </section>
      <section>
        <Recommendations />
      </section>
    </article>
  <ProductProvider>
)
```



## Current Closest Equivalent

Here's the closest I can get with our current toolset. It is debatably 
desirable for ergonomic reasons but it still has all the limitations of Context
and in-practice the scopes are implicitly coupled to a context.

### react-hoisting (fake package)
```tsx
function createScope () {
  const providers = []

  const component = ({ children }) => {
    let content = children

    for (const Provider of providers) {
      content = <Provider>{content}</Provider>
    }

    return content
  }

  const hoist = (hook) => {
    const TheContext = createContext ()

    providers.push (({ children }) => {
      const value = hook ()
      return (
        <TheContext.Provider value={value}>
          {children}
        </TheContext.Provider>
      )
    })

    return () => useContext (TheContext)
  }

  return {
    Provider: component,
    hoist,
  }
}
```

### ./provider
```tsx
import { createScope } from "react-hoisting"

export const ColorContext = createContext ("")
export const ProductIdContext = createContext ("")

export const ProductScope = createScope ()

export const ProductProvider = ({ children, id }) => {
  return (
    <ProductIdContext.Provider value={id}>
      <ProductScope.Provider>
        {children}
      </ProductScope.Provider>
    </ProductIdContext>
  )
}
```

### ./color/state.tsx
```tsx
import { ProductIdContext, ProductScope } from "./provider"

export const useSelectedColorState = ProductScope.hoist (() => {
  const productId = useContext (ProductIdContext)
  const [ selectedColorId, setSelectedColorId ] = useState ("")

  useEffect (() => {
    setSelectedColor ("")
  }, [ productId ])

  return useMemo (() => ({
    selectedColorId,
    setSelectedColorId,
  }), [ selectedColorId, setSelectedColorId ])
})
```



## Built-In API

With a built-in API we're able to infer the level of hoisting based on 
`useContext` so no Scope is needed.

### ./color/state
```tsx
export const useSelectedColorState = hoist (() => {
  const productId = useContext (ProductIdContext)
  const [ selectedColorId, setSelectedColorId ] = useState ("")

  useEffect (() => {
    setSelectedColor ("")
  }, [ productId ])

  return useMemo (() => ({
    selectedColorId,
    setSelectedColorId,
  }), [ selectedColorId, setSelectedColorId ])
})
```

### ./color/swatch.tsx
```tsx
const useColorSelected = hoist ((color: string) => {
  const { selectedColor } = useSelectedColorState ()
  return color === selectedColor
})

const useSelectColor = hoist ((color: string) => {
  const { setSelectedColor } = useSelectedColorState ()
  return useCallback (() => {
    setSelectedColor (color)
  }, [ color, setSelectedColor ])
})
```
