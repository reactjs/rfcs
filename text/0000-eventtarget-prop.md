- Start Date: 2023-03-15
- RFC PR: 
- React Issue: 

# Summary

This adds a `target` prop to every React component and a `useEventTarget` hook to use it.

# Basic example

```jsx
function ParentComponent() {
  const [text, setText] = useState("Count: 0");
  const target = useEventTarget();
  target.addEventListener("count", (evt) => {
    setText(`Count: ${evt.detail.count}`);
  });
  
  return (
    <ChildComponent target={target}>
      <h1>{text}</h1>
    </ChildComponent>
  );
}

function ChildComponent({ children, target }) {
  const [count, setCount] = useState(0);
  const onClick = (count) => {
    setCount(count);
    target.dispatchEvent(new CustomEvent("count", {
      detail: { count }
    }));
  };

  return (
    <>
      {children}
      <button onClick={() => onClick(count-1)}>-</button>
      <button onClick={() => onClick(count+1)}>+</button>
    </>
  );
}
```

# Motivation

The web is event-driven; it's time to make React be as well.

The vast majority of Web APIs involve EventTarget, including DOM elements.
It's a simple yet elegant way of sharing data without disrupting the top-down nature of the DOM.
The dispatcher has no knowledge or control over who is listening and what will be done.
The child component's only role is to signal that new information is available.

Parent-child communication without tightly coupling the two is a persistant problem in React.
Most existing solutions involve passing both the value and set function from `useState()` to the child.
Large-scale applications will often use Redux selectors and actions that operate on the same store in both parent and child.

<!--
Why are we doing this? What use cases does it support? What is the expected
outcome?

Please focus on explaining the motivation so that if this RFC is not accepted,
the motivation could be used to develop alternative solutions. In other words,
enumerate the constraints you are trying to solve without coupling them too
closely to the solution you have in mind.
-->

# Detailed design

There are two elements to this proposal: making a `target` prop available to every component the way `children` is now, and a new `useEventListener()` hook to use it.

`target` should be defined and available to every React component, regardless of whether or not the parent component has set the `target` prop.
With this change, every React component can be (mentally) thought of as the functional equivalent of a class inheriting EventTarget:
A component always has it's own `addEventListener` and `dispatchEvent` available to it. It never has to check to make sure they were given by a parent, and there are no changes to the behavior if there are no listeners.

To listen to child events, a higher-level component must use the `useEventTarget` hook.
Components call `addEventListener` to listen to events, then pass it to a component through the `target` prop.

Setting `target` should be considered equivalent to the following: `for (const [name, fn] of target) child.addEventListener(name, fn)`
To implement this, the JSX transformer will need to create an EventTarget instance, possibly using `useMemo`.
It will then need to connect the events from the `target` prop to the newly-created EventTarget instance, possibly using code like the example above.
The new EventTarget is then passed down as the `target` prop to the component along with the `children` prop.

<!--
This is the bulk of the RFC. Explain the design in enough detail for somebody
familiar with React to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.
-->

# Drawbacks

The first drawback that comes to mind is that if a component already has a `target` property, this will almost certainly break backwards-compatibility with it.

Another drawback is that since the `target` prop must always be available to any component it requires that every transformed component must keep an EventTarget instance under the hood.


<!--
Why should we *not* do this? Please consider:

- implementation cost, both in term of code size and complexity
- whether the proposed feature can be implemented in user space
- the impact on teaching people React
- integration of this feature with other existing and planned features
- cost of migrating existing React applications (is it a breaking change?)

There are tradeoffs to choosing any path. Attempt to identify them here.
-->

# Alternatives

- Do nothing. React works and there are user-land solutions for almost any problem.
- I briefly considered the [useEvent](https://github.com/reactjs/rfcs/blob/useevent/text/0000-useevent.md) hook proposed by @gaearon before realizing it was for different use-cases.
- My previous Signals and Slots RFC ([0000-signals-and-slots.md](https://github.com/Symbitic/rfcs/blob/master/text/0000-signals-and-slots.md)) was intended to solve similar problems, but it applies an outside concept instead if using the EventTarget already adopted by the web.
- User-land solutions like creating a new EventTarget and passing it. The problem is that this makes following the web optional rather than something built into React.
- Use `eventTarget` as the prop name instead of `target` as it would be less likely to already be in use.

<!--
What other designs have been considered? What is the impact of not doing this?
-->

# Adoption strategy

Once implemented

<!--
If we implement this proposal, how will existing React developers adopt it? Is
this a breaking change? Can we write a codemod? Should we coordinate with
other projects or libraries?
-->

# How we teach this

The biggest problem with teaching is that most web developers will know that EventTarget is a class, which may lead them to think that using this requires switching back to class-based components.

One way to solve this is to teach the `target` prop first, the `useEventTarget` hook second.

<!--
What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing React patterns?

Would the acceptance of this proposal mean the React documentation must be
re-organized or altered? Does it change how React is taught to new developers
at any level?

How should this feature be taught to existing React developers?
-->

# Unresolved questions

<!--
Optional, but suggested for first drafts. What parts of the design are still
TBD?
-->
