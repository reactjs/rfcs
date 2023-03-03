- Start Date: 2023-02-04
- RFC PR: (leave this empty)
- React Issue: (leave this empty)
- Author: Pranav Yadav [@Pranav-yadav](https://github.com/Pranav-yadav) 

# Summary

Better conventions for Server/Client Components, as an revision to [the v2](https://github.com/reactjs/rfcs/pull/227) of [the original proposal v1](https://github.com/reactjs/rfcs/pull/189)

<sup>Note: The v1 and v2 terms used throughout this RFC refer to the above hyperlinked RFCs only.</sup>

# Changes Since v2

- using `"use csr"` instead of `"use client"` i.e. replacing `client` with `csr`
- similarly, reserve `"use ssr"` for future use with Server only.

# Basic example

<!-- If the proposal involves a new or changed API, include a basic code example.
Omit this section if it's not applicable. -->

To mark a component as a Client Component, add `'use csr'` directive at the top of your file. This lets you use features like state and event handlers:

```js
'use csr';

import { useState } from 'react';

function Button({ children, onClick }) {
  // ...
}
```

Diff - current syntax and conventions:
```diff
+'use csr';
-'use client';

import { useState } from 'react';

function Button({ children, onClick }) {
  // ...
}
```

## `"use csr"` Directive

```js
// Parent.js
import Component from "./Component.js";
function Parent() {
  // Component is of type Reference<T> where T is the type
  // of the default export of Component.js
  // Because of this React knows that it can't render it on
  // the server and instead will leave it as a placeholder
  // to later be rendered on the client.
  return <Component />;
}
```

```js
// Component.js
"use csr";

function Component() {
  let [state, setState] = useState(false);
  return <OtherClientComponent onClick={() => setState(true)} value={state} />;
}
```

# Motivation

<!-- Why are we doing this? What use cases does it support? What is the expected
outcome?

Please focus on explaining the motivation so that if this RFC is not accepted,
the motivation could be used to develop alternative solutions. In other words,
enumerate the constraints you are trying to solve without coupling them too
closely to the solution you have in mind. -->



# Detailed design

<!-- This is the bulk of the RFC. Explain the design in enough detail for somebody
familiar with React to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here. -->



# Drawbacks

<!-- 
Why should we *not* do this? Please consider:

- implementation cost, both in term of code size and complexity
- whether the proposed feature can be implemented in user space
- the impact on teaching people React
- integration of this feature with other existing and planned features
- cost of migrating existing React applications (is it a breaking change?)

There are tradeoffs to choosing any path. Attempt to identify them here. -->



# Alternatives

<!-- What other designs have been considered? What is the impact of not doing this? -->



# Adoption strategy
<!-- 
If we implement this proposal, how will existing React developers adopt it? Is
this a breaking change? Can we write a codemod? Should we coordinate with
other projects or libraries? -->

# How we teach this

<!-- What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing React patterns?

Would the acceptance of this proposal mean the React documentation must be
re-organized or altered? Does it change how React is taught to new developers
at any level?

How should this feature be taught to existing React developers? -->


# Unresolved questions

<!-- Optional, but suggested for first drafts. What parts of the design are still
TBD? -->

