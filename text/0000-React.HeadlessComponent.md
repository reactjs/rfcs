- Start Date: 2018-08-19
- RFC PR: 
- React Issue: 

# Summary

The high level proposal is for adding a new `React.HeadlessComponent` type to React, similar to `React.PureComponent`, for the explicit purpose of tooling (and possible React optimization) to take advantage of the fact that the component will not render any elements. This is the primary reason this proposal cannot be achieved in userland.

If the high level proposal is accepted, the low level proposal for `React.HeadlessComponent`'s API is that it explicitly doesn't allow `render()` to be implemented. Instead it assumes its child is a function and calls it while passing down data as a first argument, and its instance as a second argument (allowing access to class fields, `state` and `setState`). 

Users can optionally implement a `children()` class method instead of a `render()` method, and in it you return an object which will then be included in the child function call. Returning JSX inside a `children()` class method will throw an error (see errors section below), but the object itself can contain React components (aka "[render components](https://twitter.com/swyx/status/995051406636744706)", a special case of render props).

This feature might only exist in dev mode, and compile to a regular component in prod.

# Basic example

adapted from Merrick Christensen's [Headless User Interface Components](https://www.merrickchristensen.com/articles/headless-user-interface-components/):

```js
const flip = () => ({
  flipResults: Math.random()
})

// creation of headlesscomponent
class CoinFlip extends React.HeadlessComponent {
  state = flip()
  handleClick = () => this.setState(flip)
  children() { // note this is new
    return { // return an object
      isHeads: this.state.flipResults < 0.5
    }
  }
}

// usage: note CoinFlip's `state` is the second param. `handleClick` is a class method and gets passed down too and is bound to CoinFlip's scope.
<CoinFlip>
  {({ isHeads }, parent) => (
   <>
     Flip Results: {parent.state.flipResults}
     <button onClick={parent.handleClick}>Reflip</button>
     {isHeads ? <img src=”/heads.svg” alt=”Heads” /> : <img src=”/tails.svg” alt=”Tails” />}
   </>
  )}
</CoinFlip>
```

# Motivation

> Why are we doing this? What use cases does it support? What is the expected
outcome?

> Please focus on explaining the motivation so that if this RFC is not accepted,
the motivation could be used to develop alternative solutions. In other words,
enumerate the constraints you are trying to solve without coupling them too
closely to the solution you have in mind.

React Components bundle reusable **behaviors** and **elements**. However, we often want to implement the two separately for reusability, and use composition to build up our user interfaces. [Function-as-children Render props](https://reactjs.org/docs/render-props.html#using-props-other-than-render) and [Headless components](https://medium.com/merrickchristensen/headless-user-interface-components-565b0c0f2e18) are increasingly popular design patterns to do this (even included in [the new Context API](https://reactjs.org/docs/context.html), encouraged [by Dan](https://twitter.com/dan_abramov/status/1021850499618955272) and taught by [Kent C Dodds at React Rally](https://www.youtube.com/watch?v=ii-T6HrkZFM)), however because they are entirely userland design patterns, **no guarantees exist** to ensure any particular higher order component or render prop controller will not also render elements as well as provide behavior control.

**Dev Tools**: This informal nature makes it impossible to build any tooling that leverages or even encourages this pattern. For example [React Devtools currently suffer from a pollution of higher order components](https://twitter.com/brian_d_vaughn/status/1025776259056336897) and it is difficult to have a good rule to filter out elements that don't show up in the DOM. Conversely, we can also filter -for- headless components and track only their state changes given their state changes are more likely to be significant than regular React Components.

**Testing**: Testing for Headless Components would be easier (eg with Jest) since there is guaranteed to be no render.

**Others?:** This is a bit out there but reusable behavior libraries could be searchable by their signature, similar to [Hoogle](https://www.haskell.org/hoogle/) from the haskell ecosystem. If we eliminate the possibility that the Component renders anything then we can build a lot more ecosystem tools around exploiting that one fact.

Lastly, React.HeadlessComponent would reduce some boilerplate to implementing a headless component just like PureComponent does.

# Detailed design

> This is the bulk of the RFC. Explain the design in enough detail for somebody
familiar with React to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

**This section concerns the low level API design, which is only relevant if the High level premise of this RFC is accepted.**

`React.HeadlessComponent` would be similar to `React.Component` in every way it has an optional `children` method instead of a `render`. Being headless, it will assume its child is a function and call it with data from running `children`, secondly its instance which provides class properties and state.

```js
class MyHOC extends React.HeadlessComponent {
  state = { foo: 1 }
  bumpFoo = () => this.setState(state => ({foo: state.foo + 1}))
  children() { 
    return {
      // anything that children should receive
      bar: 2
    }
  }
}

// usage
<MyHOC>
  {
    ({bar}, {bumpFoo, state, setState}) => <div>
      <div>foo: {state.foo}</div>
      <div>bar: {bar}</div>
      <button onClick={bumpFoo}>bump foo</button>
      <button onClick={() => setState({foo: 5})}>Reset</button>
    </div>
  }
</MyHOC>
```

If it affects production bundle size at all, React.HeadlessComponent could easily be compiled to a regular Component in production mode.

**Where it throws errors**

- if the child node of the HC is anything other than a function
- if the HC implements a `render()` method
- returning JSX inside `children()`

# Drawbacks

> Why should we *not* do this? Please consider:

- **implementation cost, both in term of code size and complexity**: Not much, pretty much the same cost as PureComponent
- **whether the proposed feature can be implemented in user space**: I don't think so, it has to be a part of the component type in React itself for tooling to take advantage
- **the impact on teaching people React**: It's just as optional as PureComponent is, but while PureComponent is focused on performance, HeadlessComponent is focused on DX and encouraging reusability
- **integration of this feature with other existing and planned features**: no idea. I believe the new devtools profiler Brian is working on could immediately gain from this
- **cost of migrating existing React applications (is it a breaking change?)**: not at all breaking. entirely optional, and more likely a good thing for React library authors to adopt, and React app authors to benefit from.

> There are tradeoffs to choosing any path. Attempt to identify them here.

It is a minor increase in API surface area. But it bakes in this idea.

# Alternatives

> What other designs have been considered? What is the impact of not doing this?

I haven't been able to think of any alternatives, this idea goes right to the heart of what a Component is, and what guarantees we can build around it to push folks down the pit of success.

Regarding the naming of `children()` there are alternative suggestions from [Josh Comeau on /r/reactjs](https://www.reddit.com/r/reactjs/comments/98itgz/reactheadlesscomponent_rfc/):

- `output()`
- `renderData()`

Which are also acceptable to me.

# Adoption strategy

> If we implement this proposal, how will existing React developers adopt it? Is
this a breaking change? Can we write a codemod? Should we coordinate with
other projects or libraries?

Not at all a breaking change, no coordination needed. This would be a highly desirable thing for library maintainers to use like `react-redux` and `styled-components`.

# How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing React patterns?

well the terminology of Headless Components is well accepted and it is a formalization of existing React patterns. I could also run with:

- "Renderless Components" (more neutral naming but less fun) or 
- "State-only Components" (not precise because you could technically make a HC that has no internal state but usually you do want it)
- "Behavior Components" (echos the Motivation section)

> Would the acceptance of this proposal mean the React documentation must be
re-organized or altered? Does it change how React is taught to new developers
at any level?

Nope. Just an extra section on it after React.Component.

> How should this feature be taught to existing React developers?

"If you are making reusable components that only control behavior, use this for better tooling support and less boilerplate!"

# Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?

- Whether or not HeadlessComponent needs to compile down to plain Component for production bundle size saving (i suspect it is immaterial)
- I briefly considered also passing down the HC's props to its children, but figured that would be redundant since those same props will be available in the outer scope anyway.
- I honestly don't know if there are other benefits to this I haven't thought of yet. I am quite unimaginative at this sort of thing, experienced frontend infra folks would likely be able to run with this further than I can.
- **IDE integration**: If React.HeadlessComponent existed, we could potentially provide better syntax highlighting, autocomplete and typechecking to encourage reusable headless components and reduce cognitive load for end users of libraries built with React.HeadlessComponent. But how exactly would this look like?
- Some folks may not want children to be able to see the HC's internal `state`, I only included that as a nice-to-have since it's easy to do and will very probably save boilerplate (so you can implement a useful HeadlessComponent even without `children()`). 
  - Personally I'm all for more freedom, and could even see my way to expose a `setState` to children by default (perhaps as a third argument) so that you can make up your own custom state updater's in children.
  - More freedom = less control by the component author, but more reusability guaranteed for component users. Since the second and third arguments are optional I think I can argue for this to be an advanced "use if you know how the HC works internally" feature.
