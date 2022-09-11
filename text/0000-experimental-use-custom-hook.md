- Start Date: 2022-09-11
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

In the React repository, there are a few PRs to add `experimental_use`.
- [experimental_use(promise)](https://github.com/facebook/react/pull/25084)
- [experimental_use(context)](https://github.com/facebook/react/pull/25202)

The `experimental_use` hook is a special hook that can be called conditionally.
https://twitter.com/acdlite/status/1558186162518478850

As the name implies, it can accept some other values.
This PR proposes `experimental_use(customHook)`.

# Basic example

```jsx
import { experimental_use, useState } from 'react';

const useCountState = () => useState(0);

const Component = ({ mode }) => {
  if (mode === 'simple') {
    return <div>Just nothing</div>;
  }
  const [count, setCount] = experimental_use(useCountState); // This doesn't violate the React hooks rule.
  return <div>{count}<button onClick={() => setCount(c => c + 1)}>+1</button></count>
};
```

# Motivation

There is a rule with React hooks, which is hooks can't be called conditionally.
Some considered it kind of a bad design and other frameworks chose different design decision.
It would be so nice, if we can provide a solution to it.
`experimental_use` seems like a nice fit.

# Detailed design

My understanding of React internals is very limited, but at high level
hooks implementation is just arrays. https://www.swyx.io/hooks/#not-magic-just-arrays

So, implementation-wise, it will be `Map<customHook, array>` instead of just an array.
The referential equality of `customHook` is important.

(A little more precicely, `experimental_use` can be nested, so `Map<customHook, array>` is a recursive structure forming a tree.)

A question is if this is an acceptable concept, but a similar approach is once proposed.
https://github.com/reactwg/react-18/discussions/25
> The "create" function acts as the type/key/identity for the cache.

# Drawbacks

- The implementation will be non trivial. (implementation cost, code size)
- There are two ways to use custom hooks. (mental model overhead, education cost)
- Some may not like the proposed pattern. (community split)

# Alternatives

Instead of using the custom hook function as the identity, we could use string/symbol identity.

If React is moving toward to compiler based approach,
there could be alternative solutions without introducing a new function.

# Adoption strategy

`experimental_use` is experimental, so if this can be done within the experimental phase,
we could collect feedbacks before actually releasing it.

# How we teach this

This should carefully be taught with `experimental_use`, which is a special hook that can run conditionally.

# Unresolved questions

The feasibility of the implementation is unknown.
