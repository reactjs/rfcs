- Start Date: 2023-04-10
- RFC PR: 
- React Issue: 

# Summary

This adds two props (`addEventListener` and `dispatchEvent`) to every React component and a `useEventTarget` hook to use them.

# Basic example

```jsx
function ParentComponent() {
  const [text, setText] = useState("Count: 0");
  const eventTarget = useEventTarget();
  eventTarget.addEventListener("count", (evt) => {
    setText(`Count: ${evt.detail.count}`);
  });

  return (
    <ChildComponent eventTarget={eventTarget}>
      <h1>{text}</h1>
    </ChildComponent>
  );
}

function ChildComponent({ children, dispatchEvent }) {
  const [count, setCount] = useState(0);
  const onClick = (count) => {
    setCount(count);
    dispatchEvent(new CustomEvent("count", {
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

The web is event-driven: it's time to make React be as well.

The vast majority of Web APIs involve EventTarget, including DOM elements.
It's a simple yet elegant way of sharing data without disrupting the top-down nature of the DOM.
The dispatcher has no knowledge or control over who is listening and what will be done.
The child component's only role is to signal that new information is available.

Parent-child communication without tightly coupling the two is a persistant problem in React.
Most existing solutions involve passing both the value and set function from `useState()` to the child.
Large-scale applications will often use Redux selectors and actions that operate on the same store in both parent and child.
I've experienced this myself in commercial application development, where components will often use the same `useSelector` and `dispatch` call to communicate with each other.

<!--
Why are we doing this? What use cases does it support? What is the expected
outcome?

Please focus on explaining the motivation so that if this RFC is not accepted,
the motivation could be used to develop alternative solutions. In other words,
enumerate the constraints you are trying to solve without coupling them too
closely to the solution you have in mind.
-->

# Detailed design

There are two elements to this proposal: making an `addEventListener` and `dispatchEvent` prop available to every component the way `children` is now, and a new `useEventTarget()` hook to use them.

By default, `addEventListener` and `dispatchEvent` will be empty functions: they do nothing when the component calls them. This changes when a parent component sets the `eventTarget` prop. When it does, the `dispatchEvent` and `addEventListener` props call their respective methods on the `eventTarget`.
`addEventListener` and `dispatchEvent` should be defined and available to every React component, regardless of whether or not the parent component has set the `eventTarget` prop.

With this change, every React component can be thought of (mentally) as the functional equivalent of a class inheriting EventTarget:
A component always has it's own `addEventListener` and `dispatchEvent` available to it. It never has to check to make sure they were given by a parent, and there are no changes to the behavior if there are no listeners.

To set the `eventTarget` property, a higher-level component must use the `useEventTarget` hook. When `eventTarget` has been set, the child `addEventListener` and `dispatchEvent` prop functions invoke their respective functions on the `eventTarget` given to them.

To implement this, the JSX transformer will need to do the following:
1. If the `eventTarget` prop is undefined or null, it creates `addEventListener` and `dispatchEvent` as empty functions and then expose them to the transformed component as props. They are still defined so the component can call them without throwing an exception, but there is no behavior outside the component.
2. If the `eventTarget` prop is given, it creates `addEventListener` and `dispatchEvent` as functions which call their respective methods on the EventTarget instance, then exposes them to the transformed component as props.

While the `eventTarget` prop is set by the parent, it should not be exposed to the child component.

<!--
This is the bulk of the RFC. Explain the design in enough detail for somebody
familiar with React to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.
-->

# Drawbacks

- If a component already has an `eventTarget` prop for something else, this will cause breaking changes. This may not be much of an issue since there don't seem to be any major libraries that use this.
- The implementation could require the JSX transformer to keep an EventTarget instance under the hood for every component. On the other hand, that likely wouldn't be too difficult.

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

- Do nothing. React works and there are solutions for almost any problem.
- I briefly considered the [useEvent](https://github.com/reactjs/rfcs/blob/useevent/text/0000-useevent.md) hook proposed by gaearon before realizing it was for different use-cases.
- My previous Signals and Slots RFC ([0000-signals-and-slots.md](https://github.com/Symbitic/rfcs/blob/master/text/0000-signals-and-slots.md)) was intended to solve similar problems, but it applies an outside concept instead if using the EventTarget already adopted by the web.
- User-land solutions like creating a new EventTarget and passing it. The problem with that is that it doesn't solve the problem of making the web a first-class part of React.

<!--
What other designs have been considered? What is the impact of not doing this?
-->

# How we teach this

Our goal is to make React components more web-like. The word "more" should be emphasized to convey that this is not a breaking change: it simply helps React components act more like regular DOM elements.

In addition to documenting the `useEventTarget` hook, the three related props (`addEventListener`, `dispatchEvent`, and `eventTarget`) must also be taught. Since the `children` prop isn't especially well-documented, it might be useful to create a new page "Child Props" that describes every built-in prop available to all components.

<!--
What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing React patterns?

Would the acceptance of this proposal mean the React documentation must be
re-organized or altered? Does it change how React is taught to new developers
at any level?

How should this feature be taught to existing React developers?
-->

# Unresolved questions

Should `useEventTarget` create a new instance every time it is called, or should the JSX transformer create a single instance and return it for every `useEventTarget`?

What design should be used to handle the case when an `eventTarget` prop becomes undefined later?

<!--
Optional, but suggested for first drafts. What parts of the design are still
TBD?
-->
