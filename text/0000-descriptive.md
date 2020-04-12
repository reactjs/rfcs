- Start Date: 2020-04-12
- RFC PR:
- React Issue:

# Summary

The React suspense fallback component mounts immediately when "waiting" begins and immediately
unmounts when "waiting" is complete. This creates a flashing effect of the the fallback suspense
component.

React animation libraries have solved this issue of components unmounting without any graceful
transition effects by providing apis that receive some sort of key value or boolean
that then tells the api to unmount the component with a graceful transition animation. See
[Framer Motion's Animate Presence](https://www.framer.com/api/motion/animate-presence/), 
[React Spring's useTransition](https://www.react-spring.io/docs/hooks/use-transition), and [React
Transition Group](http://reactcommunity.org/react-transition-group/transition-group).

Unfortunately, none of these graceful transitions / apis can be used by a fallback suspense
component because, currently, React suspense does not provide a prop to the fallback component
communicating to it when the fallback component is getting mounted or not.

# Motivation

* **Why is this being proposed?:** React suspense's fallback component currently will unmount
immediately after suspense is done 'waiting'. Having one component unmount and another completely
different component mount directly in its place without any transition effects creates the
appearance of an primitive ui.

* **What use cases does it support? What is the expected outcome?:**
  - If a developer can create a transition from when the fallback component unmounts and when the
   suspense's child component mounts ...

# Detailed design

This is the bulk of the RFC. Explain the design in enough detail for somebody
familiar with React to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

# Drawbacks

Why should we *not* do this? Please consider:

- implementation cost, both in term of code size and complexity
- whether the proposed feature can be implemented in user space
- the impact on teaching people React
- integration of this feature with other existing and planned features
- cost of migrating existing React applications (is it a breaking change?)

There are tradeoffs to choosing any path. Attempt to identify them here.

# Alternatives

What other designs have been considered? What is the impact of not doing this?

# Adoption strategy

If we implement this proposal, how will existing React developers adopt it? Is
this a breaking change? Can we write a codemod? Should we coordinate with
other projects or libraries?

# How we teach this

What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing React patterns?

Would the acceptance of this proposal mean the React documentation must be
re-organized or altered? Does it change how React is taught to new developers
at any level?

How should this feature be taught to existing React developers?

# Unresolved questions

Optional, but suggested for first drafts. What parts of the design are still
TBD?
