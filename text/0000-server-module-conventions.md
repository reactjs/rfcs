- Start Date: 2020-12-21
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Please watch [the React Server Components intro talk](https://reactjs.org/server-components/) and high level [React Server Components RFC](https://github.com/reactjs/rfcs/pull/188) for some background context on this RFC.

[React Server Components](https://github.com/reactjs/rfcs/pull/188) doesn't have much of an API other than integration into meta-frameworks. Instead, we propose that the main way you make choices about what runs on the server and client is defined by its file extension.

We also propose using package.json conditional exports to allow an import to fork its implementation between server and client usage.

# Basic example

## `.client.js` convention

```js
// Parent.server.js
import Component from "./Component.client.js";
function Parent() {
  // Component is of type Reference<T> where T is the type
  // of the default export of Component.client.js
  // Because of this React knows that it can't render it on
  // the server and instead will leave it as a placeholder
  // to later be rendered on the client.
  return <Component />;
}
```

When a Component with a `.client.js` extension is imported in a "React Server" environment its exports gets replaced with a special "Reference" object. This object can't be accessed directly in that file but it can be passed into React as if it was a plain component.

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

# Motivation

Traditionally, a certain code base has been mostly used for one purpose. Primarily as a server/CLI tool, or primarily as a browser app (even with SSR). In this world it's not that difficult to distinguish which environment you're working with. Especially if they're in different languages since they already have different file extensions.

Our intention is to make this line much more fluid, but tradeoffs depending on if you're on the server or client can significantly impact the user experience. We think that this is an important thing to surface, so that when you take on a dependency you know whether that is going to end up downloading on the client or not.

It is also important, as an author of a reusable component, to be able to communicate your intention about which environment this library is intended to be used. For example, if it requires direct database access or if it has waterfall data fetching.

The motivation for a way to automatically fork based on environment, is useful in cases where something idiomatically could work or partially work in either environment but would require different implementations.

We need some way to easily create "Reference" objects on the server that represents arbitrary exports on the client. We also need this to work with IDEs that auto-complete as well as type systems to track which types a component accepts.

We also would like to preserve the ability to build the client separately and parallel with the server.

# Detailed design

## `.js`, `.server.js`, `.client.js`

We envision that there will be three types of modules:

* **Server Modules** are modules that can only be imported on the server.
* **Client Modules** are modules that can only be imported on the client.
* **Shared Modules** are modules that only use a subset of features that work in either Server or Client.

We think that the most commonly used module will be a shared module since a lot of plumbing code can just work in either environment. Therefore we gave that the default file extension: `.js`

## Enforcement

In our bundler plugins we'll enforce that a `.server.js` file cannot be imported in that environment.

In our Node/server bundler plugins, we'll enforce that `.client.js` imports aren't directly importable but instead gets replaced by "References".

We'll also enforce that you can't import a `.server.js` file from a `.js` file on the server. The rationale for this is that a `.js` represents a Shared components that should be able to work in either environment. Even if it's currently only used on the server, someone should be able to import it and use it on the client.

However, we know that almost all code today is written with just a `.js` extension. These will continue to work as long as they don't recursively import something that is restricted for that environment.

If you're trying to use an existing component that uses state and has a `.js` file extension, you can either rename it to `.client.js` or add a wrapper component that re-exports it using a `.client.js`. That allows it to be used from the server.

When combined with the "exports" field, the file extension of a file is the resolved file, not the required file. This allows you to create aliases that don't require this file extension.

## Implicit Client References

As mentioned above, if a server environment tries to import a `.client.js` it instead gets a placeholder value - a Reference - that represents that export.

This doesn't only work with Components. It also works with props. This allows you to pass objects that normally not serializable to the client as long as it can be reach through a module export.

```js
// Parent.server.js
import Component from "Component.client.js";
import implA from "implA.client.js";
import {MY_CONFIG} from "LargeStaticConfig.client.js";

export default function Parent() {
  return <Component config={MY_CONFIG} impl={implA} />
}
```

When React sees a "reference" used as a component in JSX, it'll include a command for the client to early start preloading that component. When React sees the references used in this component's props, it'll also do the same for those references.

When React renders these on the client, we'll suspend until we have both the component and the references passed into the props. Instead of passing the references, we'll pass the real values from those modules.

So this works through two mechanisms: 1) Eager preloading. 2) Automatic unwrapping.

This is the common case so we want this to work out of the box and be easy to use.

## Explicit References

There are likely other needs to mimic these more explicitly. For example if you want to manually lazily load the reference on the client. We also foresee needing to pass these reference to the client and then back to the server for dependency injection purposes.

We could add some new pseudo-syntax to solve this problem. Long term we envision using something like the [Asset References](https://github.com/tc39/proposal-asset-references) proposal to expose this use case.

Once you have a reference, we can expose explicit helpers for when to preload a reference and when to unwrap it.

In theory we could use this instead of "importing" a `.client.js` file. However, since this functionality isn't widely available in tooling yet it's not practical. Additionally, we would still need different APIs to describe preloading and unwrapping. Since those are common operations, we'd need something more convenient than plain asset references anyway.

## TypeScript Support

While we use `.js` in these examples, we expect to apply the same conventions for `.jsx`, `.ts`, `.tsx` and other file extensions.

One of the motivations for reusing the `import` syntax with a file extension for this purpose is that we get TypeScript (and other tooling) support out of the box. We just pretend that it's the same environment so TypeScript can type check the callsite when a server component passes props into a client component.

However, this call should only accept serializable props even if the client component can accept non-serializable optional properties. This is an additional enforcement we're currently adding to Flow. It would be nice to have an equivalent feature in TypeScript.

Basically, TypeScript would have to know about this reference type, what it unwraps to, and that JSX calls on this reference type only accepts props given certain constraints of how we define serializability.

## Conditional Exports

Conditional exports is a fairly new feature respected by [Node.js](https://nodejs.org/api/packages.html#packages_conditional_exports), [Webpack](https://webpack.js.org/guides/package-exports/#conditional-syntax) and others. This allows you to fork an implementation for various environments.

We propose that whether you're using Node.js runtime or building a server bundle, that those module resolutions add the `"react-server"` condition.

This allows a package to fork its implementation based on whether it's used on the server or the client. For example, a data framework might support simple data fetching on the server but on the client it can support optimistic updates or fine grained invalidation in a client environment. Even React itself uses this technique to exclude stateful Hook exports on the server.

One strange aspect is that Server-side Rendering environments, should not use `"react-server"`. This is a bit confusing but SSR isn't a "server environment" in this world. It's a "client emulator". It emulates running Client components and emits their HTML and later re-executes them on the client. It's likely we'll try to rebrand SSR help clarify this distinction.

It's not best practice to fork client components based on if they use SSR or not because the hydration should use the same path. If there is a fork in behavior, then it should be forked on the client as well during hydration. That needs a separate dynamic flag that's different than this.

It might seem confusing when you look at it hard enough, but we've found that when you're working on a product you mostly spend your time thinking of .server or .client disinction and these provide the right intuition.

A possible downside of this approach is that you can't run SSR and a React Server environment in the same module graph. You can however build two separate bundles and run them in the same Node.js VM. However, we expect most people to want to use two separate Node.js instances for parallelization purposes anyway and possibly even putting the SSR instance close to the "edge" and the React Server instance close to the data.

Why not also add `"react-client"` add as condition though? Client components are the default today. The issue is that the existing environments and ecosystem won't have that configured. If you added something like `"exports": {".": {"react-client: "..."}}` to a bundle, it wouldn't work in those environments. In the future sometime we could add this and discourage its use until it has propagated to all environments, but for now it's not useful. It can be expressed as the lack of `"react-server"` instead.

These conditions can be combined with `"node"` and `"browser"` conditions. For example, we plan on having a browser environment for debugging "react-server". This technique can be used for example to provide an `"fs"` polyfill only in a browser debugging environment for `"react-server"` but forbid it in a React client.

## Bundler Integration

React Server needs deep bundler integration with the runtime and changes to bundling heuristics, which is out of scope for this RFC.

However, in all configurations the bundler needs to be able to know about which `.client.js` files to include in the bundle. The Server Environment and Client Environment is bundled separately, by default the client bundle wouldn't include any of these files since they're not reachable from any of the client components included in the shell of the app. We must therefore tell the bundler to include them.

We expect production builds to build the server first and then use this tree information as input to build the client. However, during development it is useful to be able to just rebuild the client separately. In some scalable configurations, it's also helpful to not use the reachability of the graph but instead every build every file on the file system in parallel. By using a file extension we can build the client bundle but simply just telling it to build all `.client.js` files it can find.

# Drawbacks

The main controversial part of this proposal is likely to be that we change the typical behavior of importing a JS file to mean something different. We mostly do this for pragmatic reasons - there is not idiomatic way to do this today.

It also means that we indirectly coopt the meaning of these file extensions in the larger ecosystem. If other npm libraries use `.client.js` or `.server.js` files and those end up using in a React app, they might not work. This would then force others to comply to these conventions even if they're not directly related to React.

File extensions don't really compose well with other extensions. You can't have two dimensions that both require the extension to be at the end. However, allowing the extension anywhere in the filename makes it even more likely to have collisions with other uses.

Adding a file extension to files and to most imports adds noise to your code. If our thesis of a lot of components being expressible as Shared components is incorrect, we might end up with a lot of `.client.js` file extensions in the ecosystem.

# Alternatives

We don’t necessarily have to have a convention around this split at all. For example, we could automatically use a compiler to split out files based on reachability. However, in early experiments with this it became very difficult to reason about the resulting performance when you don’t have a mental model around where the boundary is. A small change up top in a branch can pull a whole tree down with it into the client.

Instead of using file extensions, we could use some special syntax for referencing the client from the server. Then we’d use the reachability of the graph to determine which bundle things end up in. This effectively happens with the `.js` extension in this proposal. However, this would suffer some of the predictability issues and doesn’t communicate intent of how a file is supposed to be used. For example, it might technically work on the client but pulls in a lot of code/data that it probably shouldn’t.

We could possibly use file extensions in combination with only explicit syntax such as [Asset References](https://github.com/tc39/proposal-asset-references) or [Import Assertions](https://github.com/tc39/proposal-import-assertions), but that syntax is not yet available, widely supported by tooling and not convenient enough for the common scenario.

Instead of file extensions, we could use a package.json configuration for whether it should run on the server or client. That’s not granular enough. We expect a lot of switching back and forth even in a single package. It would be a hassle to configure it for every file.

Similarly, we considered using folders instead of file extensions, but you often want to use pairs of Shared and Client components. E.g. a wrapper that can be executed on the server and a small implementation detail on the client. Moving files is also generally harder than renaming, and breaks up colocation. File extensions are also frequently used for special purposes both in Node and Webpack already - but not folders.

# Adoption strategy

The approach we're taking will need strong bundler integration to perform well. Therefore, we'll have specific bundler plugins provided by the runtime that they support. However, ideally you shouldn't need anything else. For example, existing TypeScript and Lint rules can still apply because we just expands on existing conventions.

There is some infrastructure that likely needs deeper support - such as CSS-in-JS libraries need to be designed to be able to ship CSS without the JS.

The idea is that you start using a `.server.js` component at the root and see which is the first component that fails to build because it uses state. Then you rename that to `.client.js` or add a `.client.js` wrapper that re-exports if you can't rename it. We also have automatic scripts that can detect and do this conversion.

The approach is opt-in and doesn't break existing client code. The only exception is that `.server.js` files would error on the client if you enable our plugins.

# How we teach this

If you need to useState, useEffect or dependencies that only work on the client, rename the file to `.client.js`.

If you need direct access to databases or other server resources, rename to `.server.js`.

Split the file into two separate files if you don't need all of it to execute in either environment.

# Unresolved questions

For explicit references, it's unclear which syntax we should use. Either we can try to adopt an early stage proposal or use some intermediate pseudo-syntax.

It's not clear which file-extensions this should apply to. `.js` is not enough because of other languages like `.ts` but does non-code like `import x from "foo.client.png"` import a Reference?
