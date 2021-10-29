- Start Date: 2021-10-30
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Inject prop to all components or specific components

# Basic example

Sometimes we import some modules into every component's file ex: i18n translation module or history module or tracker/logger module

# Motivation

Motivation about this RFS is improvement bundle size by injecting prop, also simply component's file by reducing any unnecessary imports

# Detailed design

Imagine we have something like this for injecting prop to all components

```js
import React, { inject } from "react";
import history from "history-package-or-module";
import t from "i18n-package-or-module";

inject({
  // history helper methods
  push: (...args) => history.push(...args),
  // translation helper method
  t: (...args) => t(...args),
  // GA helper method
  tracker: (...args) => GA.dataLayer.push(...args),
});
```

So we can use this method without importing its modules

```jsx
function Component({ push, t, tracker }) {
  const handleSubmit = () => {
    push("/somewhere");
    tracker("user is going to somewhere");
  };

  return <button onClick={handleSubmit}>{t("submit")}</button>;
}
```

Also we can have `excludes` or `includes` as option for just inject to specific component by name like this:

```js
inject(
  {
    // history helper methods
    push: (...args) => history.push(...args),
    // translation helper method
    t: (...args) => t(...args),
    // GA helper method
    tracker: (...args) => GA.dataLayer.push(...args),
  },
  {
    includes: ["Home", "SignUp", "SignIn"], // all of them is component names
  }
);
```

# Drawbacks

I'm not sure about implementation cost, I think this is not hard to learn and also this is no need for any migration plan

# Alternatives

Nothing, or maybe I found nothing

# Adoption strategy

Lucky this hasn't any breaking change

# How we teach this

The term about that is `Injection` also all developer familiars to this concept

# Unresolved questions

Nothing
