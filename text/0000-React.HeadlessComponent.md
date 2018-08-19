- Start Date: 2018-08-19
- RFC PR: 
- React Issue: 

# Summary

React.HeadlessComponent is a new sibling to React.PureComponent that explicitly doesn't allow `render()` to be implemented. Instead it assumes its child is a function and calls it while passing down its state (and perhaps any other class fields declared on it). 

Users can also implement a `children()` class method that looks like a `render()` method, but in it you return an object which will then be included in the child function call. Returning JSX inside a `children()` class method will throw an error, but the object itself can contain React components.

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
  {({ handleClick, isHeads }, state) => (
   <>
     Flip Results: {state.flipResults}
     <button onClick={handleClick}>Reflip</button>
     {isHeads ? (
       <div>
         <img src=”/heads.svg” alt=”Heads” />
       </div>
     ) : (
       <div>
         <img src=”/tails.svg” alt=”Tails” />
       </div>
     )}
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

React Components bundle reusable **behaviors** and **elements**. However, we often want to implement the two separately for reusability, and use composition to build up our user interfaces. [Function-as-children Render props](https://reactjs.org/docs/render-props.html#using-props-other-than-render) and [Headless components](https://medium.com/merrickchristensen/headless-user-interface-components-565b0c0f2e18) are increasingly popular design patterns to do this (even included in [the new Context API](https://reactjs.org/docs/context.html) and taught by [Kent C Dodds at React Raly](https://www.youtube.com/watch?v=ii-T6HrkZFM)), however because they are entirely userland design patterns, no guarantees exist to ensure any particular higher order component or render prop controller will not also render elements as well as provide behavior control.

**Dev Tools**: This informal nature makes it impossible to build any tooling that leverages or even encourages this pattern. For example [React Devtools currently suffer from a pollution of higher order components](https://twitter.com/brian_d_vaughn/status/1025776259056336897) and it is difficult to have a good rule to filter out elements that don't show up in the DOM. Conversely, we can also filter -for- headless components and track only their state changes given their state changes are more likely to be significant than regular React Components.

**IDE integration**: If React.HeadlessComponent existed, we could potentially provide better syntax highlighting, autocomplete and typechecking to encourage reusable headless components and reduce cognitive load for end users of libraries built with React.HeadlessComponent. 

**Testing**: Testing for Headless Components would be easier (eg with Jest) since there is guaranteed to be no render, but the components would still have access to all the lifecycle methods of a regular component.

**Others?:** This is a bit out there but reusable behavior libraries could be searchable by their signature, similar to [Hoogle](https://www.haskell.org/hoogle/) from the haskell ecosystem. If we eliminate the possibility that the Component renders anything then we can build a lot more ecosystem tools around exploiting that one fact.

Lastly, React.HeadlessComponent would reduce some boilerplate to implementing a headless component just like PureComponent does.

# Detailed design

> This is the bulk of the RFC. Explain the design in enough detail for somebody
familiar with React to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

`React.HeadlessComponent` would be similar to `React.Component` in every way except it throws an error if `render` is implemented. In its place, it can (optionally) implement a `children` method that expects an object to be returned. This object will be spread together with its state when calling its child function.

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
    ({bar, bumpFoo}, state) => <div>
      <div>foo: {state.foo}</div>
      <div>bar: {bar}</div>
      <button onClick={bumpFoo}>bump foo</button>
    </div>
  }
</MyHOC>
```

Edge case: Can't think of any.

If it affects production bundle size at all, React.HeadlessComponent could easily be compiled to a regular Component in production mode.

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

# Adoption strategy

> If we implement this proposal, how will existing React developers adopt it? Is
this a breaking change? Can we write a codemod? Should we coordinate with
other projects or libraries?

Not at all a breaking change, no coordination needed.

# How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing React patterns?

well the terminology of Headless Components is well accepted and it is a formalization of existing React patterns.

> Would the acceptance of this proposal mean the React documentation must be
re-organized or altered? Does it change how React is taught to new developers
at any level?

Nope. Just an extra section on it after React.Component.

> How should this feature be taught to existing React developers?

"If you are making reusable components that only control behavior, use this for better tooling support!"

# Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?

- Whether or not HeadlessComponent needs to compile down to plain Component for production bundle size saving (i suspect it is immaterial)
- I briefly considered also passing down the HC's props to its children, but figured that would be redundant since those same props will be available in the outer scope anyway.
- I honestly don't know if there are other benefits to this I haven't thought of yet. I am quite unimaginative at this sort of thing, experienced frontend infra folks would likely be able to run with this further than I can.
