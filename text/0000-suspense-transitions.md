- Start Date: 2020-04-12
- RFC PR:
- React Issue:

# Summary

The React suspense fallback component mounts immediately when "waiting" begins and immediately
unmounts when "waiting" is complete. This creates a flashing effect of the fallback suspense
component.

React animation libraries have solved this issue of components unmounting without any graceful
transition effects by providing apis that receive some sort of key value or boolean
that then tells the api to unmount the component with a graceful transition animation. See
[Framer Motion's Animate Presence](https://www.framer.com/api/motion/animate-presence/), 
[React Spring's useTransition](https://www.react-spring.io/docs/hooks/use-transition), and [React
Transition Group](http://reactcommunity.org/react-transition-group/transition-group).

Unfortunately, none of these graceful transitions / apis can be used by a fallback suspense
component because, currently, React suspense does not communicate when it is "waiting" and when
it is done "waiting".

# Motivation

* **Why is this being proposed?:** React suspense's fallback component currently will unmount
immediately after suspense is done "waiting". Having one component unmount and another completely
different component mount directly in its place without any transition effects creates the
appearance of a primitive ui.

* **What use cases does it support? What is the expected outcome?:** If a developer can create a
 transition between the fallback component unmounting and suspense's child component mounting,
this will create a more smoother ui.

# Detailed design

### Assumptions

In this design proposal, I am assuming that React's `suspense` receives a boolean that tells it when
to render the fallback component and when to render the children. Without being familiar
with the suspense source code, I have created a pseudo example, where `isWaiting` refers to if
the Suspense component is "waiting".

```jsx
const Suspense = ({ children, fallback, isWaiting }) => isWaiting ? fallback : children;
```

### Proposal 1

**[CodeSandbox](https://codesandbox.io/s/suspense-rfc-v1-zic4k?file=/index.js)**

This proposal works by wrapping React's suspense with a parent component that provides
transition effects. In order for this to be accomplished, this parent component needs to be aware
of the "isWaiting" state:

```jsx
import ReactDOM from "react-dom";
import React from "react";
import { motion, AnimatePresence } from "framer-motion";

// This isn't how concurrent mode works, just some pseudo code
const ConcurrentMode = ({ children, isWaiting }) => React.cloneElement(children, { isWaiting });

// Some pseudo code simulating Suspense
const Suspense = ({ children, fallback, isWaiting }) => isWaiting ? fallback : children;

// Provides animations
const AnimationWrapper = ({ isWaiting, children }) => (
  <AnimatePresence exitBeforeEnter>
    <motion.div
      animate="enter"
      exit="exit"
      initial="initial"
      key={isWaiting}
      variants={...}
    >
      {React.cloneElement(children, { isWaiting })}
    </motion.div>
  </AnimatePresence>
);

const App = () => (
  <ConcurrentMode>
    <AnimationWrapper>
      <Suspense fallback={<FallbackComponent />}>
        <ChildrenComponent />
      </Suspense>
    </AnimationWrapper>
  </ConcurrentMode>
);

export default App;
```

**Drawbacks:**
The downside of this is that the parent component, in the example above, `AnimationWrapper`,
needs to be aware of the "isWaiting" state *in addition* to React's suspense component. I'm
currently not sure sure how easy this would be to accomplish given I am not as familiar with the
suspense source code, but it would seem to break the api (?).

### Proposal 2 - recommended

**[CodeSandbox](https://codesandbox.io/s/suspense-rfc-v2-jlzur?file=/index.js)**

This proposal works by adding a new prop to React's suspense called 'Wrapper'. This wrapper
essentially accomplishes what was outlined in Proposal 1 but is rendered as part of the
suspense api. It also provides the "isWaiting" state as a prop to the fallback and children
components so that those components can access those states if needed for any other transition
tooling.

```jsx
import ReactDOM from "react-dom";
import React from "react";
import { motion, AnimatePresence } from "framer-motion";

// This isn't how concurrent mode works, just some pseudo code
const ConcurrentMode = ({ children, isWaiting }) => React.cloneElement(children, { isWaiting });

// Some pseudo code simulating Suspense
const Suspense = ({ children, fallback, isWaiting }) => isWaiting ? fallback : children;

// Proposed Suspense api
const ProposedSuspense = ({ children, fallback, isWaiting, Wrapper }) => {
  if (Wrapper) {
    return (
      <Wrapper isWaiting={isWaiting}>
        {isWaiting
          ? React.cloneElement(fallback, { isWaiting })
          : React.cloneElement(children, { isWaiting })}
      </Wrapper>
    );
  }
  return isWaiting
    ? React.cloneElement(fallback, { isWaiting })
    : React.cloneElement(children, { isWaiting });
};

// Provides Animations
const AnimationWrapper = ({ isWaiting, children }) => (
  <AnimatePresence exitBeforeEnter>
    <motion.div
      animate="enter"
      exit="exit"
      initial="initial"
      key={isWaiting}
      variants={animationConfig}
    >
      {React.cloneElement(children, { isWaiting })}
    </motion.div>
  </AnimatePresence>
);

// Pseudo code for how suspense currently is implemented
const SuspenseWithCurrentApi = () => (
  <ConcurrentMode>
    <Suspense fallback={<h2>fallback component</h2>}>
      <h2>child component</h2>
    </Suspense>
  </ConcurrentMode>
);

// Pseudo code for how suspense would be implemented via proposal
const SuspenseWithProposal = () => (
  <ConcurrentMode>
    <ProposedSuspense
      fallback={<h2>fallback component<h2/>}
      Wrapper={AnimationWrapper}
    >
      <h2>child component</h2>
    </ProposedSuspense>
  </ConcurrentMode>
);
```

*Drawbacks:*
The downsides to this proposal is that before, the suspense component was a rather
straightforward api; by now extending that api to support some sort of wrapper component or hoc,
this might increase the learning curve (but possibly for only those looking this kind of solution -
this may not need to be part of the introductory suspense docs)

# Drawbacks

Drawbacks have been detailed above. In addition to increasing the suspense code size, complexity,
and documentation required, those not inherently looking for this solution (adding transition
effects to suspense) might not immediately understand how to implement the api if not documented
well. Ideally, this request should not cause any breaking changes.

Also, when documenting examples, the docs might have to use another library such as Framer Motion,
React Spring, React Transition Group, etc to exemplify how these transitions would work. This may be
less than ideal since these apis (as every api) are always changing, and the docs might have to
keep track with the api from time to time. If needed, hopefully a library can be picked with an
easy to understand mental model that can be ok if it gets locked to a certain api version.

# Alternatives

Two designs have been proposed, but by no means are these the only possible mental models that
allow for transitions for suspense.

# Adoption strategy

If accepted, this should use an adoption strategy similar to react hooks given that it should
not break any current implementations of suspense and should ideally be part of some sort of
initial experimental roll out.

# How we teach this

Some additional documentation to [suspense](https://reactjs.org/docs/concurrent-mode-suspense.html)
with some sort of header like "Adding transition effects to Suspense". It may be required to use
a third party library to document this example with some explicit transition effects.
