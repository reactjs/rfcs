- Start Date: 2022-10-11
- RFC PR: https://github.com/reactjs/rfcs/pull/227
- React Issue: (leave this empty)

# Summary

This is a new take on [the original v1 of this proposal](https://github.com/reactjs/rfcs/pull/189). See the discussion there for deeper context of the reason for the changes made.

Please watch [the React Server Components intro talk](https://reactjs.org/server-components/) and high level [React Server Components RFC](https://github.com/reactjs/rfcs/pull/188) for some background context on this RFC.

[React Server Components](https://github.com/reactjs/rfcs/pull/188) doesn't have much of an API other than integration into meta-frameworks. Instead, we propose that the main way you make choices about what runs on the server and client is defined by a directive.

We also propose using package.json conditional exports to allow an import to fork its implementation between server and client usage.

# Changes Since v1

- No more file extension conventions.
- A `"use client"` directive at the top of a file defines that it's a boundary between server and client.
- Server and Client Components no longer have to explicitly state so. Everything is implicitly Shared until proven otherwise, and then the build errors.
- `import "server-only"` and `import "client-only"` can be used to poision a module so it can only be imported in that environment.
- TypeScript Convention for enforcing serializable boundaries.

# Basic example

## `"use client"` Directive

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
"use client";

function Component() {
  let [state, setState] = useState(false);
  return <OtherClientComponent onClick={() => setState(true)} value={state} />;
}
```

When a Component with a `"use client"` directive (similar to `"use strict"`) is imported in a "React Server" environment its exports gets replaced with a special "Reference" object. This object can't be accessed directly in that file but it can be passed into React as if it was a plain component.

React, together with the bundler, knows how to send this reference to the client. On the client it's rendered as a Client component. The real file never actually gets imported on the server.

## `"react-server"` conditional exports

```js
// package.json
{
  ...,
  "exports": {
    ".": {
      "react-server": {
        "browser": "./debugger-polyfill.server.js",
        "node": "./native-impl.server.js"
      },
      "default": {
        "browser": "./browser-impl.client.js",
        "node": "./node-polyfill.client.js"
      }
    }
  }
}
```

A package can choose to fork its exports depending the environment it executes in across two dimensions.

The `"react-server"` condition applies only React Server Component environments. However, these can execute either in a `"browser"` during development for debugging purposes, or it can execute in a `"node"` environment in production.

React Client Component environments are the default and they typically execute in a `"browser"` environment but they can also execute in a `"node"` environment for SSR purposes.

Note: This condition can only be used for conditional `imports` within the project itself.

Possible addition: A possible addition to this can be to also support the `"server-only"` condition as a less React specific alternative that could be adopted by other frameworks that also choose this strategy.

# Poisoned Imports

The `"react-server"` condition can be used to generate poisoned imports that can only be used by in one environment or another.

I've already published these packages:

`server-only/package.json`
```
"exports": {
  ".": {
    "react-server": "./empty.js",
    "default": "./index.js"
  }
}
```

In this case `index.js` throws an error, so if you import this from a Client Component - even in SSR - you get an error.

`client-only/package.json`
```
"exports": {
  ".": {
    "react-server": "./error.js",
    "default": "./index.js"
  }
}
```

In this case `error.js` throws an error, so if you import this from a Server Component - you get an error.

This lets you mark certain packages as only working in one environment or another using:

```
import "server-only";
```
or
```
import "client-only";
```

This can help enforce things statically with early error messages before you do a mistake. Especially since these conventions has such a strong uncanny valley. It can also enforce security or performance by ensuring that certain keys are only available to the server or that data fetching only happens from Server Components.

The nice thing about this convention is that it works outside of Server Components compatible environments by ensuring that `server-only` throws in those. It also lets you configure your own custom error messages for libraries since it's just a convention.

# TypeScript Convention

Types between the Server and Client boundary mostly just work because they behave the same as if there was no boundaray. One limitation is that if those types includes a non-serializable type - according to the React Server Components specification, you get an error.

The correct way to model this is to model Client References for exports at the boundaries and then have the callsite of JSX accept a Client Reference as a component and enforce the serialization at the callsite. This ensures that you can still pass client references from server to client in other forms than JSX. In fact, this is how Flow already implements it and it could be implemented in TypeScript too. However, this requires a bit more involved type checking from the TypeScript side.

It's also not necessarily the best enforcement because it doesn't prevent a component from adding a non-serializable prop as inputs unless it can also type check against all possible usages. Which might not be best for error messages and not be viable if you're publishing a shared component that's used in other repos.

Therefore, we propose another convention to files with a `"use client"` directive:

The conventions apply if a module exports a function that is identified as a React Component, according to some definition specified by the [React Compiler](https://reactjs.org/blog/2022/06/15/react-labs-what-we-have-been-working-on-june-2022.html#react-compiler) and the [Rules of Hooks Lint rule](https://reactjs.org/docs/hooks-rules.html). Basically if it contains a capital name and contains JSX or Hooks.

In that case, such function's arguments must all be serializable according to the Serialization rules of [React Server Components](https://github.com/reactjs/rfcs/pull/188) (basically JSON + Cycles, Binary Data, Promises, JSX etc).

This can be enforced by a framework using code gen, a TypeScript plugin or just built-in to TypeScript. This doesn't need any additional information from the graph to be enforced.

The implications of this is also that you shouldn't add the `"use client"` directive to all Client Components. Only the ones intended to be used directly by the Server. Because indirect ones are still allowed to have non-serializable props.

# Drawbacks

The main drawback of this new approach is that we further the uncanny valley. It's hard to tell when you jump into any given file whether that is going to get used as a Shared, Client and Server component. That's the big downside of the whole Server Components design that we're taking a bet that it's worth learning this one thing to unlock its possibilities. To help combat this, we'll also propose that, as a convention Server Components are mostly written as "Async Functions" form instead of "Hooks Functions". More on that soon. While Shared and Client Components would be written as "Hooks Functions". This distinction is not enforced though.

Additionally, since it's not enforced, it's difficult for tooling to tell if a file is a Shared, Client or Server Component. The only thing where you can tell that for sure is at the boundaries that are annotated. Otherwise you need to inspect the dependency graph. A lot of early errors might only be detected by a bundler that has the graph available to it.

It also places constraints on the bundler. To find all Client Components to put in the client bundle, you now need to do a partial pass to parse them. It's not enough to just bundle all `.client.js` files. However, since in practice, most bundling has to follow the dependency graph to tree shake files that are not used you probably have to do that anyway.

Because of the `"react-server"` export condition, the correct way to implement a module graph requires duplicating module resolution on the server. Not necessarily duplicating the bytes in each module but the resolution has to happen twice. If module A imports module B which is forked based on `"react-server"`, then there has to be two copies of module A. One for SSR and one for RSC. Since `"react"` itself is forked on `"react-server"` it means that any file that imports `"react"` needs to be initialized twice too. Naively you end up needing two separate bundles for SSR and for RSC. This can be good for the client navigation case since it's only have to load the RSC bundle then but worse for the SSR case. This is probably the most controversial part of the design.

The benefit is that we can now statically and early error when you try to import something itself imports `useState` or `useEffect` and give better early error messages for easy mistakes.

Another benefit is that this kind of fork is needed for the [React Compiler](https://reactjs.org/blog/2022/06/15/react-labs-what-we-have-been-working-on-june-2022.html#react-compiler) to be able to optimize SSR separately from RSC. E.g. it doesn't make sense to lower things to "concatenating pieces of HTML" for generating the RSC payload. So if you want to take advantage of this, you'll need something like it anyway.

Right now dereferencing a Client Reference wouldn't be a TypeScript error.

```
// server-file.tsx;
import {Foo} from 'client-file.tsx';

Foo.hello();
```

This wouldn't be a type error because TypeScript doesn't know that this is a boundary from a Server to a Client component. If it was Server-to-Server or Client-to-Client it wouldn't matter. To really know that, it would have to be informed by the Graph. A smart framework could do that but it would be difficult for TypeScript to do that out of the box.

# Alternatives

See the discussion in [the v1 proposal](https://github.com/reactjs/rfcs/pull/189) to see some of the alternatives that we've considered.
