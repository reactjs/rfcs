- Start Date: 2018-09-20
- RFC PR:
- React Issue:

# Summary

A new API `React.Children.toFlatArray` will traverse not only the direct children
but will also go into `Fragments` and recursively traverse their children as well
so that the resulting array does not contain any `Fragments`.

# Basic example

```jsx harmony
import React, { Fragment, Children } from 'react'

function TabsContainer({ children }) {
  console.log(Children.toFlatArray(children)) // [<Tab/>, <Tab/>, <Tab/>] instead of [<Tab/>, <Fragment />]
}

function App() {
  return (
    <TabsContainer>
       <Tab/>
       <Fragment>
         <Tab/>
         <Tab/>
       </Fragment>
    </TabsContainer>
  )
}
```

# Motivation

There has been a need to operate on sets of sibling elements without a wrapper DOM element. 
This need has been addressed by introducing `<React.Fragment>` which allows passing around
lists of elements. 

The original documentation for `Fragments` also stated that they will be transparent 
for the APIs in the `React.Children` namespace, i.e.:

> If children is a keyed fragment or array it will be traversed: 
> the function will never be passed the container objects.

This design is arguably convenient: there are likely only a few cases where the developer 
of the component would like to differentiate between receiving a `Fragment` as a child
and receiving children of the `Fragment` directly.

Contrary to this documentation the APIs in the `React.Children` namespace do not traverse 
children of the `Fragments`. There have been reports about this (
[1](https://github.com/facebook/react/issues/11859) 
[2](https://github.com/facebook/react/issues/12662)
[3](https://github.com/facebook/react/issues/13677)
) but apparently React developers consider this an intentional behavior.

At the moment there is no elegant solution to traverse the children of an element
while treating `Fragments` as transparent containers. An example use case for this would be
a tabs container inspecting its children tabs where the list of tabs is dynamic 
and several of them are wrapped in a conditionally present `Fragment`.
Arguably every such pair of container and element components will require a way to access
the direct descendants of the container, wrapped in `Fragment` or not.

One way to solve this would be to change the existing behaviour 
of the APIs in the `React.Children` namespace to reflect the original documentation. 
This solution has downsides of being a potentially breaking change of existing APIs
and of preventing further evolution of `Fragments`. Its upsides are smaller API surface
and fewer surprises for developers who use i.e. `React.Children.count` and suddenly break it
by wrapping the children in a `Fragment`.

The proposed solution is to lean on existing API `React.Children.toArray`
and to provide a similar API `React.Children.toFlatArray`. 
The latter would treat the `Fragments` as transparent containers 
and would not include them in the output while including their children.
Other APIs could be left intact since all of them could be easily re-implemented for this array.
The upsides are the absence of breaking changes and the potential to extend `Fragments`
with new features. The downsides are larger API surface and potentially surprising behaviour 
of the rest of the APIs in the `React.Children` namespace.

# Detailed design

A new API `React.Children.toFlatArray` is added to convert 
the `this.props.children` opaque data structure to a flat array of elements
with keys assigned to each child. 

The difference with `React.Children.toArray` is that if any of the elements 
in `this.props.children` is a `Fragment` then this `Fragment` is not added 
to the resulting array, instead its direct children are added. 
If this `Fragment` contains another `Fragment` then the inner `Fragment` 
is expanded in the same way recursively.

An example use case:

```jsx harmony
import React, { Fragment, Children } from 'react'

function TabsContainer({ children }) {
  const tabs = Children.toFlatArray(children)
  const titles = tabs.map(tab => tab.props.title)
  return (
    <Fragment>
      {titles.join(', ')}
      {children}
    </Fragment>
  )
}

function SettingsForm(props) {
  const { basic } = props
  return (
    <TabsContainer>
       <Tab id="base" />
       {basic || (
         <Fragment>
           <Tab id="advanced" />
           <Tab id="developer" />
         </Fragment>
       )}
    </TabsContainer>
  )
}
```

# Drawbacks

- larger API surface due to the new API
- the rest of the APIs in the `React.Children` namespace keep their potentially surprising behaviour

# Alternatives

1) Instead of creating a new API a new parameter can be added to the `React.Children.toArray`.
   This is less visible in the code and less explicit compared to the `React.Children.toFlatArray`.
2) Existing behaviour of the APIs in the `React.Children` namespace could be changed 
   to treat `Fragments` as transparent containers. This is a potentially breaking change which also
   hampers the potential further development of `Fragments`.
3) A separate small npm package could handle this task. This will be much less visible to developers,
   and it will have to transform `keys` once again. 

# Adoption strategy

On discovering problematic behaviour of existing APIs developers will refer to React’s documentation 
and will find the solution there. The same scenario applies to the developers of other
React-based libraries. 

# How we teach this

The term `flat` is universally accepted for this purpose across frameworks and programming languages.
No special explanation or presentation is necessary beside saying that both arrays and `Fragments`
are flattened. New API does not change how React is taught.

The documentation of the other APIs in the `React.Children` namespace will have to me modified
to hint that flattening `Fragments` requires the use of this new API. The documentation on `Fragment`
might benefit from hinting at this as well.

The existing React developers will refer to React’s documentation on discovering 
the “problematic” APIs behaviour and will find the solution in the documentation. 
