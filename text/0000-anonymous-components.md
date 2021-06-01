- Start Date: 2021-02-09
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

This RFC introduces a new pattern/primitive, the `Anonymous` component (and the `anonymous` function as a layer of syntatic sugar). Final naming is TBD, userland proof of concept [here](https://github.com/lewisl9029/react-anonymous), implementation is a [1 liner](https://github.com/lewisl9029/react-anonymous/blob/master/src/index.js).

Essentially, the anonymous components pattern aims to give users the option to use hooks in a way that doesn't force them to introduce indirection (i.e. restructure their components' internal branching, hoist variables to top of component functions, introduce new components that have to be named, prop drilled, have new types defined, etc) for the sole purposes of adhering to the rules of hooks.

The goal of the RFC at this point in time is only to unblock adoption of the [userland implementation](https://github.com/lewisl9029/react-anonymous) through a modification of the official React hooks linter rule. Any changes to official docs/recommendations can come if/when the userland implementation gains enough traction to be considered sufficiently battle tested.

# Basic example

Examples are slightly contrived to fit in a variety of use cases, but hopefully it showcases all the new possibilities adequately. I've left some inline comments sprinkled throughout to help focus attention and provide additional context.

```js
import * as React from 'react'
// Using my proof of concept in userland: https://github.com/lewisl9029/react-anonymous
import { Anonymous } from '@lewisl9029/react-anonymous'

const TodoList = ({ 
  loading, 
  todos 
}: { 
  loading: boolean, 
  todos: TodoType[],
}) => {
  if (loading) {
    // Call hooks within an early return
    return (
      <Anonymous>{() => {
        const { startAt } = React.useContext(LoadingAnimationTimingContext)
        return <Loading animationStartAt={startAt} />
      }}</Anonymous>
    )
  }

  // Call hooks after an early return
  return (
    <Anonymous>{() => {
      const [todoId, setTodoId] = React.useState()

      return (
        <>
          {todoId ? (
            // Call hooks within an inline branching path
            <Anonymous>{() => (
              <DetailsModal
                onClose={React.useCallback(() => setTodoId(undefined), [])}
                todo={todos[todoId]}
              />
            )}</Anonymous>
          ) : null}
          <Todos>
            {todos.map((todo) => (
              // Call hooks within a mapping function
              <Anonymous>{() => {
                const [isComplete, setIsComplete] = React.useState(false)

                return (
                  <Todo
                    todo={todo}
                    onClick={React.useCallback(() => setTodoId(todo.id), [])}
                    isComplete={isComplete}
                    onComplete={React.useCallback(() => setIsComplete(true), [])}
                  />
                )
              }}</Anonymous>
            ))}
          </Todos>
        </>
      )
    }}</Anonymous>
  )
}
```

For contrast, here's an equivalent implementation without the anonymous component pattern (again using comments to offer context):

```js
import * as React from 'react'

// There are probably better names for this component.
//
// Either way, note that we've had to waste brain cycles thinking
// about the name of a component that we didn't have to name before.
//
// Even though it has 1 usage and isn't being reused, and could
// have been inlined if it weren't for the rules of hooks.
const StatefulTodo = ({
  todo,
  setTodoId,
}: {
  // Note that these types used to be fully infered from the
  // upstream types in the closure.
  //
  // Now we must specify them due to the indirection added.
  // Imagine if there were a dozen more of these.
  todo: TodoType,
  setTodoId: (todoId: string) => void,
}) => {
  const [isComplete, setIsComplete] = React.useState(false)

  return (
    <Todo
      todo={todo}
      // Note how linter can no longer infer `setTodoId` never changes.
      onClick={React.useCallback(() => setTodoId(todo.id), [setTodoId])}
      isComplete={isComplete}
      onComplete={React.useCallback(() => setIsComplete(true))}
    />
  )
}
// Note how we could no longer follow the implementation
// in natural reading/execution order from top to bottom.
//
// We just read the implementation of StatefulTodo before
// we had the chance to build up context on its _only_ usage.

const TodoList = ({
  loading,
  todos,
}: {
  loading: boolean
  todos: TodoType[]
}) => {
  // Note the that this is used only in the loading branch
  const { startAt } = React.useContext(LoadingAnimationTimingContext)
  // And this is only used in the loaded branch
  const [todoId, setTodoId] = React.useState()
  // Imagine if we needed to useMemo something expensive used only in 1 branch.
  // How often are we incurring the cost of that useMemo in both branches?
  // How often do we introduce another layer of indirection to optimize?

  // Note the additional distance introduced between definition and usage
  // compared to before, where we had 0 distance.
  //
  // Also note how downright useless this name is (adds no new information).
  const onClose = React.useCallback(() => setTodoId(undefined), [])

  if (loading) {
    return <Loading animationStartAt={startAt} />
  }

  return (
    <>
      {todoId ? (
        <DetailsModal
          onClose={onClose}
          todo={todos[todoId]}
        />
      ) : null}
      <Todos>
        {todos.map((todo) => (
          // Note that we've had to introduce a new named component
          // and prop drill to adhere to rules of hooks.
          //
          // Imagine if this component needed a dozen more props.
          //
          // Have you ever had to refactor out a component like
          // this just to introduce a hook call, and in the process
          // having to drill dozens of new props and define dozens
          // of new types?
          <StatefulTodo todo={todo} />
        ))}
      </Todos>
    </>
  )
}
```

Hopefully this was enough to pique your interest and demonstrate how this pattern can drastically reduce the amount of friction involved with using hooks on a day to day basis.

# Motivation

## Hooks usage ergonomics are severely compromised by rules of hooks

Part of the [original motivation](https://reactjs.org/docs/hooks-intro.html#its-hard-to-reuse-stateful-logic-between-components) for React Hooks was to reduce indirection compared to earlier userland patterns like render props and higher-order components:

> If you’ve worked with React for a while, you may be familiar with patterns like render props and higher-order components that try to solve this. But these patterns require you to restructure your components when you use them, which can be cumbersome and make code harder to follow.

React Hooks in its current iteration doesn't fully accomplish this design goal, as noted in the [original list of drawbacks](https://github.com/reactjs/rfcs/blob/master/text/0068-react-hooks.md#drawbacks):

> - The “Rules of Hooks”: in order to make Hooks work, React requires that Hooks are called unconditionally. Component authors may find it unintuitive that Hook calls can't be moved inside if statements, loops, and helper functions.
> - The “Rules of Hooks” can make some types of refactoring more difficult. For example, adding an early return to a component is no longer possible without moving all Hook calls to before that conditional.

In practice, transitioning from render props and HOCs to React Hooks involved trading the various forms of indirection necessitated by those patterns for a different form of indirection necessitated by the rules of hooks, which often forces users to create extra layers of components and prop drilling and hoist variable definitions far away from their usage sites for the sole purpose of ensuring hook calls remain at the top level of component functions.

Overall, judging from the incredible rate of hooks adoption, this was a tradeoff that the community was very much willing to accept in exchange for the myriad of benefits hooks provided over the previous patterns, but that doesn't mean we shouldn't try to explore ways to improve usage ergonomics even further. 

The anonymous components pattern introduced in this RFC has the potential to bring hooks closer to the original vision of an approach for sharing logic between components that doesn't force users to introduce indirection (i.e. restructure their components) when there is otherwise no compelling reason to do so.

## Users, not libraries, should dictate when to introduce indirection

On a more fundamental level, I believe that the choice to introduce indirection or not should be left to the case-by-case judgement of the user, not something that should be forced upon them by arbitrary constraints of the APIs they use. 

Control over indirection (control over granularity of componentization, variable declarations/hoisting, orchestration of control flows, etc in this case), is one of the most powerful tools we have for managing complexity and cognitive overhead when reading and working with code.

The anonymous component pattern introduced in this RFC aims to restore this control over indirection to the hands of users, something that has been often deprived from them in countless instances by the rules of hooks ever since its introduction.

## Opening up additional use cases for hooks

Even more interestingly, the anonymous component pattern also opens up the possibility for a new set of use cases for hooks, where previously the excessive levels of indirection introduced by the rules of hooks made usage ergonomics prohibitively poor. 

My most prominent use case for this is an experimental [useStyles](https://github.com/lewisl9029/use-styles) CSS-in-JS hook meant to provide styles to elements without forcing indirection through named styled components at every step of the way (think of it as a runtime-only version of `@emotion/babel-plugin`'s [css prop](https://emotion.sh/docs/css-prop)). 

In a world where the rules of hooks restrictions remain rigid, a CSS-in-JS hook like this has very limited benefit over existing `styled.div`-style APIs of emotion and styled-components, since the rules of hooks would be violated for a large number of potential call sites (i.e. `<div className={useStyles({/* */}, [])} />`) if called directly with no indirection, so would have to be refactored into an isolated "styled component" anyways. The anonymous components pattern can be applied to make these usages valid without any signficant refactoring.

I'm hopeful that more use cases for React hooks like this will become viable once we're able to alleviate this forced indirection problem.

# Detailed design

## The `Anonymous` component

The `Anonymous` component is a component that delegates the implementation of its entire render function to the user through its `children` render prop. 

```js
export const Anonymous = ({ children }) => children()
```

## How it works

Users can render this component with arbitrary implementation to call hooks at any level of nesting/branching/mapping within a tree of React elements, while keeping consistent hook call ordering thanks to the `Anonymous` component acting as a distinct component for hooks call ordering at every level. 

Here's what usage of the `Anonymous` component looks like in practice in the most trivial case: 

```js
const Example = () => {
  return (
    <Anonymous>
      {() => { 
        return useMemo(() => someExpensiveFunction(), [])
      }}
    </Anonymous>
  )
}
```

At first sight, this might look like a violation of the rules of hooks on calling within a callback.

In reality, the hooks end up getting called at the top level of the `Anonymous` component's render function thanks to the `children` render prop, so no violation of the rules of hooks will have occurred at runtime.

Effectively, what actually runs looks more like this:

```js
const Anonymous = () => {
  return useMemo(() => someExpensiveFunction(), [])
}

const Example = () => {
  return <Anonymous />
}
```

## Usage with linting

The [official linting rule for React hooks](https://www.npmjs.com/package/eslint-plugin-react-hooks) is not able to recognize these hook calls as valid, however, and will still treat them as violations. So we'll need to make an exception in the linting rule for the anonymous component pattern, as I've done in [this fork](https://github.com/facebook/react/compare/master...lewisl9029:support-render-hooks-in-rule-of-hooks).

This was actually my main motivation for writing up this RFC in hopes of getting official blessing for the pattern, since the lack of support in the linter rule is by far the biggest practical impediment for this pattern to gain widespread adoption in the real world (more thoughts related to this [here](#react-hooks-and-static-analysis)).

## Syntactic sugar

We also currently offer an `anonymous` function as syntactic sugar on top of the `Anonymous` component.

```js
export const anonymous = (children, { key } = {}) => React.createElement(Anonymous, { children, key })
```

Here's what it looks like in usage:

```js
const Example = () => {
  return anonymous(() => { 
    return useMemo(() => someExpensiveFunction(), [])
  })
}
```

Contrasting that with the component API:

```js
const Example = () => {
  return (
    <Anonymous>
      {() => { 
        return useMemo(() => someExpensiveFunction(), [])
      }}
    </Anonymous>
  )
}
```

We can see that it has a more compact vertical footprint by 5 lines, as formatted by the current version of prettier, and is significantly less burdensome to type. In the [basic example](#basic-example) at the top, I ended up formatting manually to make the vertical real-estate usage comparable to the function version, in order to present it in the best possible light. In practice, I use the function API almost exclusively over the component API.

## An unintuitive edge case

In the function API, we must also accept the React `key` as an arg to offer a workaround for this edge case where the same `Anonymous` component ends up getting rendered in both branches:

```js
isOpen
  ? anonymous(() => useMemo(() => "yes", []), { key: "yes" })
  : anonymous(() => useMemo(() => "no", []), { key: "no" })
```

When `isOpen` changes, React will rerender the same `Anonymous` component with a different children prop instead of unmounting/remounting a separate component for the other branch if we don't have a `key` to distinguish between them. 

This can result in bugs in cases like above where the memo value will not update when a different branch gets rendered, and worse, in cases where the brances have different numbers of hook calls, will result in runtime exceptions.

We take the `key` as part of an options object instead of accepting it directly to allow for future extension without introducing breaking changes.

In the component API, we don't need a special API for this since users can pass in a key to the component directly like they always did before:

```js
isOpen
  ? <Anonymous key="yes">{() => useMemo(() => "yes", [])}</Anonymous>
  : <Anonymous key="no">{() => useMemo(() => "no", [])}</Anonymous>
```

This is one of the most unintuitive parts to using this pattern, and I've raised it as a [drawback](#drawbacks), and as [an open question](#branching-between-top-level-anonymous-components) to outline my ideas to resolve this in both the short and long term, and to get thoughts from the community.


# Drawbacks

Here are some drawbacks that I've thought about so far (will attempt to add to this as new issues are discovered in comments):

## A new pattern to teach/learn

This is yet another new pattern to teach, and makes the rule of hooks more nuanced than it already is, and thus potentially harder to teach as well. 

Although this pattern also happens to be completely opt-in, so can be learned and applied on an as-needed basis when good use cases are found, or never learned and applied at all for those who don't encounter these use cases or don't find value in it.

## Potential for bugs/exceptions when branching between top-level anonymous components

React [optimizing away anonymous components](#an-unintuitive-edge-case) at the top of 2 branching paths as the same component being rendered with different props can result in bugs and runtime exceptions if not supplied with different keys, which can be very error prone and unintuitive.

## Introduces additional complexity for static analysis

This pattern creates more complexity in the linter rule implementation. 

I have implemented the necessary changes in [this fork](https://github.com/facebook/react/compare/master...lewisl9029:support-render-hooks-in-rule-of-hooks), but it could very well be missing edge cases. 

It may create more complexity for the [react-refresh babel plugin](https://github.com/facebook/react/blob/master/packages/react-refresh/src/ReactFreshBabelPlugin.js) as well, though I haven't looked into it in detail yet. 

Same applies to any other form of static analysis the React team may be planning to introducing in the future. More thoughts on this one in the [additional considerations section](#react-hooks-and-static-analysis).

## Removes artificial limits to component function sizes

It is possible to use this pattern to create what some could consider _unreasonably_ large component functions, than would otherwise be possible with the current rules of hooks, since the rules of hooks can act as a forcing function that limits component function size in many cases. 

Though in my opinion, this forcing function adds net negative value as it removes too many degrees of freedom over when to add/remove indirection from the hands of users. This was discussed extensively in the [motivation section](#motivation) and is core to the value prop.

## Visual noise and boilerplate

This pattern can add extra lines and extra levels of indentation in component functions when used, which can add up to a significant level of visual noise. This could be partially alleviated with shorthand APIs that make simple usages more likely to be inlined, but not entirely. 

Although the status quo of introducing new components to adhere to rules of hooks can end up being much more boilerplate heavy in many cases, so it's remains to be seen whether this will result in a net reduction or increase in _overall volume_ of boilerplate. 

Though I would posit that the kind of boilerplate introduced by the anonymous component pattern (extra Anonymous components/anoymous function calls and the resulting extra layers of indentation and lines) is a much less insidious form of boilerplate compared to the kinds of boilerplate resulting from adhering to the rules of hooks today (extra components with props/types definitions, prop drilling/spreading, hoisting of branch-specific logic, etc), as it doesn't introduce indirection or force us to manually synchronize multiple sources of truth in our code.

## Performance implications

This pattern can introduce a significant number of `Anonymous` component nodes, which can make debugging more cumbersome, and may have performance implications for reconciliation due to a larger component tree. 

Though the pattern also happens to enable users to more easily call hooks conditionally inside branches that were previously called unconditionally at the top level, so it's not immediately obvious whether this would lead to a net performance improvement or degradation at scale in real world scenarios. 

To be safe, I'd operate under the assumtion that the net effect on performance will be negative. Even still, I feel the overall improvement to developer ergonomics more than justifies this in most use cases. The beauty of the fully opt-in nature of the pattern, however, is that we can still micro-optimize away the pattern for any bottlenecks we identify through profiling.

# Alternatives

## Naming

This pattern went through several naming iterations:

- I started with "render hooks", reflecting a combination between render props and hooks.
- Then "boundary components", as it creates a component "boundary" inline at which users can call hooks.
- Finally "anonymous components", which I like the best so far, as I feel the semantics & mental model it generates aligns very well with the usage pattern and how things work under the hood.

Happy to take other suggestions as well, of course.

## Component API vs function API

I'm still not entirely settled on whether to recommend the component API or the function one for mass adoption. Here are some of the tradeoffs I've been pondering about:

- The function version only takes up 2 extra lines, and 1 level of indent in a prettier formatted codebase, compared to 4 extra lines and 2 levels of indent for the component version. This is why the function API is the one I currently use, and what I recommend as the default in the userland library, though once this pattern gains enough adoption, we could possibly petition prettier to format the `Anonymous` component differently.
- The function version may look more foreign inside a component function as it's not JSX and as a result may be more difficult to teach. 
- The component version doesn't need any special APIs for passing in props like the React `key`, as it's meant to be rendered like any other component.

In the current state, the function API is a _lot_ more ergonomic to use, but its advantages over the component API may be short lived if the pattern can gain adoption and influence projects like prettier to implement special case support for it. But at the same time, lack of support in prettier may hinder early adoption if we went with the component API, creating a chicken-and-egg situation.

## Making the linter rule more flexible

Because the linter rule is the only thing stopping this pattern from gaining adoption as a purely userland solution, we could potentially add some configuration to the linter rule to allow for this pattern, instead of endorsing and promoting the pattern officially.

In fact I would probably prefer to start with this approach to unblock more widespread adoption, so we can work out any potential unforeseen issues in userland, and then eventually have the React team endorse it officially when/if it becomes sufficiently battle tested. More in the [following section](#adoption-strategy).

## Turning every basic DOM element and React.Fragment into optional Anonymous components

This is a far more radical idea that I haven't had the chance to fully flesh out yet, but imagine if every basic DOM element accepted the same `children` function API as an Anonymous component, and can act as a new component boundary for inline hooks usage like the Anonymous component.

```js
const Example = ({ loading }) => {
  if (loading) {
    // Capital D since this JSX can't render this otherwise, but 
    // with native support in react-dom, that can change
    return <Div>{() => useMemo(() => 'Loading', [])}</Div>
  }

  return <Div>{() => useMemo(() => 'Loaded', [])}</Div>
}
```

This can work if the basic elements were implemented as something like this:

```js
// Need a separate component to execute children in due to 
// rule of hooks on branching
const AnonymousDiv = ({ children }) => <div>{children()}</div>

const Div = (props) => {
  if (typeof props.children === 'function') {
    return <AnonymousDiv {...props} />
  }

  return <div {...props} />
}
```

I wouldn't recommend we jump straight into this solution head-first, but it certainly has some interesting benefits around boilerplate reduction over the separate Anonymous component that I'd like to explore further in userland.

# Adoption strategy

There are no breaking changes involved in this proposal as it's a completely new pattern. The pattern can be adopted in a grassroots manner as people find compelling use cases for it.

However, the current iteration of the linter rule is a blocker to adoption since it incorrectly assumes hook calls inside anonymous components are in violation of the rules of hooks.

We can approach adoption in a two step process:

1. Modify the linting rule to recognize the pattern as an exception to unblock userland adoption, but without officially endorsing it.

2. Once the pattern gains significant traction and is sufficiently battle-tested, then start releasing official blog posts and docs to endorse the pattern and teach it.

# How we teach this

As mentioned in the alternatives section, I've gone through a few iterations of naming, and am currently happy with the latest terminology of "anonymous components" for the pattern, and "Anonymous" for the name of the component/function APIs. 

The name draws parallels to anonymous functions, which is a useful analogue for building the mental model. With the introduction of this pattern, we now have both named and anonymous components, and anonymous ones don't have to be explicitly created before they are used, and can be used inline within other component functions, unlike named components. 

Though it's not a perfect analogy, as under the hood, the anonymous component is really just the same named component that's being re-used over and over again with different implementations, rather than created on the spot at usage as is the case with anonymous functions. 

Always happy to take suggestions on alternatives that haven't been considered yet.

I feel a good way to teach this could be to introduce it in a standalone guide describing its various use cases, and linking to the guide in places in the existing docs where we discuss rules of hooks by mentioning how anonymous components can help. 

My experience in this area is definitly lacking though, so would love to get ideas/thoughts from the more seasoned educators on the React team and in the community. Though this is something we can defer to when we approach step 2 in the adoption process.

# Unresolved questions and additional considerations

## Implications for concurrent mode/server components

Does this pattern pose any implications for concurrent mode and/or server components that I may have missed? Nothing obvious has popped up in my mind, but I can't confidently say I have a great grasp of the nuances there.

## Branching between top-level anonymous components

As mentioned in [an earlier section](#an-unintuitive-edge-case), using the pattern can result in bugs and exceptions when two branches of a component render an anoymous component at the to level like the example below:

```js
isOpen
  ? anonymous(() => useMemo(() => "yes", []))
  : anonymous(() => useMemo(() => "no", []))
```

The current workaround offered involves relying on the user to supply a different key between branches to make React recognize that these branches need to be treated as different components to be unmounted/remounted in subsequent renders, which is definitely not ideal since it is much too reliant on human diligence and thus very error prone.

One obvious solution is to add an auto-fixable linting rule to detect these cases and reminder users to supply keys to each branch.

Though the solution I'm personally more interested in exploring in the longer term involves potentially implementing the Anonymous component as a distinct first-class primitive (rather than just a regular userland component) that React can treat differently in its reconciliation process in order to work around edge cases like this if at all possible. In the process, we may discover opportunities to further improve performance and/or debugging experience over the userland version.

## Use cases outside of hooks

I'd love some help from the community to brainstorm use cases for this pattern outside of hooks.

Here's one that I've discovered involving contexts:

```js
import * as React from 'react'
import anonymous from "@lewisl9029/react-anonymous";

const themeContext = React.createContext()

const Example = () => {
  return (
    <themeContext.Provider value={{ foreground: 'black', background: 'white' }}>
      {anonymous(() => {
        // Calling useContext(themeContext) outside of anonymous would result in undefined.
        // 
        // Since only components downstream from the component where the Provider 
        // is rendered can read from the Provider.
        const theme = React.useContext(themeContext)
        return (
          <span className={
            useStyles(
              () => ({ 
                color: theme.foreground, 
                backgroundColor: theme.background 
              }), 
              [theme.background, theme.background]
            )}
          >
            themed text
          </span>
      })}
    </themeContext.Provider>
  )
}
```

## React, hooks, and static analysis

As mentioned throughout the RFC, the only reason why this pattern couldn't have gained adoption as a purely userland solution was due to the fact that it was blocked by the official linter rule.

The React team has expressed interest in pursuing further investments in static analysis in the form of linter rules and compilers as a means to further improve hooks usage ergonomics and reduce room for error.

Regardless of whether or not this RFC gets accepted, I'd like for the React team to see this RFC as a cautionary tale on how _relying on static analysis can act as an enormous point of friction, and dramatically limit the scope and feasibility of userland experimentation and innovation_, which has been an integral part of how React has evolved to this point. 

Hooks itself evolved from and improved upon userland solutions to similar problems (HoCs and render props). But in my view, the fact that it practically demands to be used in conjunction with a linter rule (and eventually a more elaborate compiler) to maintain correctness and ergonomics is a fundamental design flaw that future evolutions of the API should attempt to eliminate, and not something to double down on.

If the React team values maximizing the degrees of freedom for those of us in userland to experiment, and eventually see those experiments gain mass adoption and inspire further innovation, I would love to see a shift in focus towards designing runtime APIs that are inherently ergonomic and promote correct usage, without requiring the aid of static analysis.
