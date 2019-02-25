-   Start Date: 2019-02-25
-   RFC PR: (leave this empty)
-   React Issue: (leave this empty)

# Summary

-   Make functional components implement `React.memo` by default
-   Add `React.always` that flags the functional component to be treated as it is currently
-   Add a warning to `React.memo` when it is used without defining `arePropsEqual`.

# Basic example

```js
import { always } from 'react';

function Button(props) {
    // Component code
}

// for users who wish to opt out of memoization
export default always(Button);
```

Only import `memo` if `arePropsEqual` is implemented.

```js
function arePropsEqual(prevProps, nextProps) {
    return prevProps.color.id === nextProps.color.id;
}

// warning is given is arePropsEqual is undefined
export default memo(Button, arePropsEqual);
```

# Motivation

With hooks becoming more widely accepted, there will be many cases where the component tree could be made up of entirely functional components.

In such cases, `React.memo` is needed to prevent children from rerendering.

However, it can become easy to forget to memoize a functional component.

In most cases, I believe reconciliation to be a more expensive operation than diffing props.

# Detailed design

`React.always` returns a component type that tells React "render the inner type and do not bail out".

`React.always` accepts any valid component type as the first argument. This ensures that it can safely wrap an import without being concerned about its implementation details.

# Drawbacks

-   Breaking change and potentially slow.

# Alternatives

-   Call it something different like `unmemo`.

# Adoption strategy

This is a breaking change, therefore any functional componenets that rely on the render never bailing out will need to implement `React.always`. Components that implement `React.memo` will not have their implementation changed, however if `arePropsEqual` is not implemented, a warning will be given in developer mode.

# How we teach this

Many other UI libraries/frameworks advertise this sort of optimization as being selling point. I believe for most users this is expected to be done already.

# Unresolved questions

Optional, but suggested for first drafts. What parts of the design are still
TBD?

-   Will memoization by default be slower on average than reconciliation?
