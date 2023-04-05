# RFC: React Server Components

* Start Date: 2020-12-21
* RFC PR: https://github.com/reactjs/rfcs/pull/188
* React Issue: (leave this empty)

> ⚠️ **NOTE: We strongly recommend [watching our talk introducing Server Components](https://reactjs.org/server-components) before reading this RFC.**

# Table Of Contents

* [Summary](#summary)
* [Changes Since v1](#changes-since-v1)
* [Basic Example](#basic-example)
* [Motivation](#motivation)
    * [Zero-Bundle-Size Components](#zero-bundle-size-components)
    * [Full Access to the Backend](#full-access-to-the-backend)
    * [Automatic Code Splitting](#automatic-code-splitting)
    * [No Client-Server Waterfalls](#no-client-server-waterfalls)
    * [Avoiding the Abstraction Tax](#avoiding-the-abstraction-tax)
    * [Distinct Challenges, Unified Solution](#distinct-challenges-unified-solution)
* [Detailed Design](#detailed-design)
    * [Simplified Loading Sequence](#simplified-loading-sequence)
    * [Refetch Sequence](#update-refetch-sequence)
    * [Capabilities & Constraints of Server and Client Components](#capabilities--constraints-of-server-and-client-components)
    * [Sharing Code Between Server and Client](#sharing-code-between-server-and-client)
    * [Conventions For Server/Client Components](#conventions-for-serverclient-components)
    * [Open Areas of Research](#open-areas-of-research)
* [Drawbacks](#drawbacks)
* [Alternatives](#alternatives)
* [Adoption Strategy](#adoption-strategy)
* [How We Teach This](#how-we-teach-this)
* [Credits And Prior Art](#credits-and-prior-art)
* [FAQ](#faq)

# Summary

> ⚠️ **NOTE: We strongly recommend [watching our talk introducing Server Components](https://reactjs.org/server-components) before reading this RFC.**

This RFC discusses an upcoming feature for React called **Server Components**. Server Components allow developers to build apps that span the server and client, combining the rich interactivity of client-side apps with the improved performance of traditional server rendering:

* Server Components **run only on the server and have zero impact on bundle-size**. Their code is never downloaded to clients, helping to reduce bundle sizes and improve startup time.
* Server Components **can access server-side data sources**, such as databases, files systems, or (micro)services.
* Server Components **seamlessly integrate with Client Components** — ie traditional React components. Server Components can load data on the server and pass it as props to Client Components, allowing the client to handle rendering the interactive parts of a page.
* Server Components **can dynamically choose which Client Components to render**, allowing clients to download just the minimal amount of code necessary to render a page.
* Server Components **preserve client state when reloaded**. This means that client state, focus, and even ongoing animations aren’t disrupted or reset when a Server Component tree is refetched. 
* Server Components are **rendered progressively and incrementally stream rendered units of the UI to the client**. Combined with Suspense, this allows developers to **craft intentional loading states** and **quickly show important content** while waiting for the remainder of a page to load.
* Developers can also **share code between the server and client**, allowing a single component to be used to render a static version of some content on the server on one route and an editable version of that content on the client in a different route. 

# Changes Since v1

- [We've replaced the `.server.js` / `.client.js` filename convention with a `'use client'` directive](https://github.com/reactjs/rfcs/pull/227).
- [We've added native `async / await` support to Server Components](https://github.com/reactjs/rfcs/pull/229).

# Basic Example

This example renders a simple Note with a title and body. It renders a non-editable view of the Note using a Server Component and optionally renders an editor for the Note using a Client Component (a traditional React component). First, we render the Note on the server.

```javascript
// Note.js - Server Component

import db from 'db'; 
// (A1) We import from NoteEditor.js - a Client Component.
import NoteEditor from 'NoteEditor';

async function Note(props) {
  const {id, isEditing} = props;
  // (B) Can directly access server data sources during render, e.g. databases
  const note = await db.posts.get(id);
  
  return (
    <div>
      <h1>{note.title}</h1>
      <section>{note.body}</section>
      {/* (A2) Dynamically render the editor only if necessary */}
      {isEditing 
        ? <NoteEditor note={note} />
        : null
      }
    </div>
  );
}
```

This example illustrates a few key points:

* This is “just” a React component -- it takes in props and renders a view. There are some constraints of Server Components - they can’t use state or effects, for example - but overall they work as you’d expect. More details are provided below in [Capabilities & Constraints of Server and Client Components](#capabilities--constraints-of-server-and-client-components).
* Server Components can directly access server data sources such as a database, as illustrated in (B). This is implemented via a [generic mechanism that supports arbitrary asynchronous data sources with `async` / `await`.](https://github.com/reactjs/rfcs/pull/229) 
* Server Components can hand off rendering to the client by importing and rendering a “client” component, as illustrated in (A1) and (A2) respectively. Client Components contain a [`'use client'` directive at the top of the file](https://github.com/reactjs/rfcs/pull/227). Bundlers will treat these imports similarly to other dynamic imports, potentially splitting them (and all the files they import) into another bundle according to various heuristics. In this example, `NoteEditor.js` would only be loaded on the client if `props.isEditing` was true. 

In contrast, Client Components are the typical components you’re already used to. They can access the full range of React features: state, effects, access to the DOM, etc. The name “Client Component” doesn’t mean anything new, it only serves to distinguish these components from Server Components. To continue our example, here’s how we can implement the Note editor:

```javascript
// NoteEditor.js - Client Component

'use client';

import { useState } from 'react';

export default function NoteEditor(props) {
  const note = props.note;
  const [title, setTitle] = useState(note.title);
  const [body, setBody] = useState(note.body);
  const updateTitle = event => {
    setTitle(event.target.value);
  };
  const updateBody = event => {
    setTitle(event.target.value);
  };
  const submit = () => {
    // ...save note...
  };
  return (
    <form action="..." method="..." onSubmit={submit}>
      <input name="title" onChange={updateTitle} value={title} />
      <textarea name="body" onChange={updateBody}>{body}</textarea>
    </form>
  );
}
```

*This looks like a regular React component because it is:* Client Components are just regular components. The [`'use client'` directive](https://github.com/reactjs/rfcs/pull/227) at the top marks this component (and any modules it imports) as code that must execute on the Client.

An important consideration is that when React renders the results of Server Components on the client it preserves the state of Client Components that may have been rendered before. Specifically, React merges new props passed from the server into existing Client Components, maintaining the state (and DOM) of these components to preserve focus, state, and any ongoing animations.

# Motivation

Server Components address a number of challenges in React that we’ve seen across a wide range of apps. Initially we searched for targeted solutions to these problems as this can often lead to a simpler solution. However we weren’t satisfied with the approaches this led us to. The fundamental challenge was that React apps were client-centric and weren’t taking sufficient advantage of the server. If we could allow developers to easily leverage their server more, we could solve all of these challenges and provide a more powerful approach to building apps, small or large. 

These challenges fall into two main buckets. First, we wanted to make it easier for developers to fall into the “pit of success” and achieve good performance by default. Second, we wanted to make it easier to fetch data in React apps. If you have used React before you have likely wished for at least some of the following capabilities:

### Zero-Bundle-Size Components

Developers constantly have to make choices about using third-party packages. Using a package to render some markdown or format a date is convenient for us as developers, but it increases code size and hurts performance for our users. On the other hand, rewriting these features ourselves is time-consuming and error-prone. While advanced features such as tree-shaking can help somewhat, we still end up having to ship *some* extra code to the user. For example, today rendering our Note example with Markdown might require over 240K of JS (over 74K gzipped individually). 

Note that *this is just one set of libraries you might use* and there may be smaller or larger alternatives. Even if you prefer alternative libraries that are only X bytes, *your users still have to download those X bytes*. The point of this example is not to pick on any particular library but to demonstrate that *using libraries is helpful as developers but increases bundle size and can hurt application performance*:

```javascript
// NOTE: *before* Server Components

import marked from 'marked'; // 35.9K (11.2K gzipped)
import sanitizeHtml from 'sanitize-html'; // 206K (63.3K gzipped)

function NoteWithMarkdown({text}) {
  const html = sanitizeHtml(marked(text));
  return (/* render */);
}
```

However, many parts of an application aren’t interactive and don’t need full data consistency. For example, “detail” pages often show information about a product, user, or other entity and don’t need to update in response to user interaction. The Note example here is a perfect example.

Server Components allow developers to render static content on the server or during the build, taking full advantage of React’s component-oriented model and freely using third-party packages while incurring zero impact on bundle size. If we render the above example as a Server Component, we can use the *exact same code* for our feature but avoid sending it to the client - a code savings of over 240K (uncompressed):

```javascript
// Server Component === zero bundle size

import marked from 'marked'; // zero bundle size
import sanitizeHtml from 'sanitize-html'; // zero bundle size

function NoteWithMarkdown({text}) {
  // same as before
}
```

### Full Access to the Backend

One of the most common pain points when writing React apps is deciding how to access your data — or where to store your data in the first place. There are many great approaches to data-fetching with React, and many great databases and data stores to choose from. However, each of these approaches have some challenges. A common theme is that developers have to expose additional endpoints to power their UI, or use existing endpoints that weren’t always designed with that UI in mind. We wanted a solution that would both make it easier for anyone to get started *and* that could tame complexity in larger apps too. 

For example, if you’re creating a new app (or even your very first app!) and aren’t sure where to store your data, you can start with your file system:

```javascript
import fs from 'fs';

async function Note({id}) {
  const note = JSON.parse(await fs.readFile(`${id}.json`));
  return <NoteWithMarkdown note={note} />;
}
```

More sophisticated apps can similarly take advantage of direct backend access to use databases, internal (micro)services, and other backend-only data sources:

```javascript
import db from 'db';

async function Note({id}) {
  const note = await db.notes.get(id);
  return <NoteWithMarkdown note={note} />;
}
```

### **Automatic Code Splitting**

If you’ve worked with React for a while you may be familiar with the concept of [code splitting](https://reactjs.org/docs/code-splitting.html), which allows developers to break their application into smaller bundles and send less code to the client. Common approaches are lazily loading a bundle per route and/or lazily loading different modules based on some criteria at runtime. For example, applications may lazily load different code (in different bundles) depending on the user, content, feature flags, etc.:

```javascript
// PhotoRenderer.js
// NOTE: *before* Server Components

import { lazy } from 'react';

// one of these will start loading *when rendered on the client*:
const OldPhotoRenderer = lazy(() => import('./OldPhotoRenderer.js'));
const NewPhotoRenderer = lazy(() => import('./NewPhotoRenderer.js'));

function Photo(props) {
  // Switch on feature flags, logged in/out, type of content, etc:
  if (FeatureFlags.useNewPhotoRenderer) {
    return <NewPhotoRenderer {...props} />; 
  } else {
    return <OldPhotoRenderer {...props} />;
  }
}
```

Code splitting can be very helpful in improving performance, but there are two main limitations to existing code-splitting approaches. First, developers have to remember to do it at all, replacing regular import statements with `React.lazy` and dynamic imports. Second, this approach delays the point at which the application can start loading the selected component, offsetting some of the benefit of loading less code!

Server Components address these limitations in two ways. First, they make code splitting automatic: Server Components treat all imports of Client Components as potential code-split points. Second, they allow developers to make the choice of which component to use much earlier, on the server, so that the client can download it earlier in the rendering process. The net effect is that Server Components let developers focus more on their app and write natural code, with the framework optimizing delivery of the app by default:

```javascript
// PhotoRenderer.js - Server Component

// one of these will start loading *once rendered and streamed to the client*:
import OldPhotoRenderer from './OldPhotoRenderer.js';
import NewPhotoRenderer from './NewPhotoRenderer.js';

function Photo(props) {
  // Switch on feature flags, logged in/out, type of content, etc:
  if (FeatureFlags.useNewPhotoRenderer) {
    return <NewPhotoRenderer {...props} />;
  } else {
    return <OldPhotoRenderer {...props} />;
  }
}
```

### No Client-Server Waterfalls

A common cause of poor performance occurs when applications make sequential requests to fetch data. For example, one pattern for data fetching is to initially render a placeholder and then fetch data in a `useEffect()` hook:

```javascript
// Note.js
// NOTE: *before* Server Components

function Note(props) {
  const [note, setNote] = useState(null);
  useEffect(() => {
    // NOTE: loads *after* rendering, triggering waterfalls in children
    fetchNote(props.id).then(noteData => {
      setNote(noteData);
    });
  }, [props.id]);
  if (note == null) {
    return "Loading";
  } else {
    return (/* render note here... */);
  }
}
```

When both a parent and child component use this approach, however, the child can’t start loading any data until the parent component finishes loading its data. There are some positive aspects of this pattern though. One upside of fetching data in individual components is that it allows an application to fetch exactly the data it needs and avoid fetching data for parts of the UI that aren’t rendered. We wanted to find a way to avoid sequential round trips *from the client* while still avoiding over-fetching of data that wouldn’t be used.

Server Components allow applications to achieve this goal by moving sequential round trips to the server. The problem isn’t really the round trips, it’s that they are from client to server. By moving this logic to the server we reduce the request latency and improve performance. Even better, Server Components allow developers to continue fetching the minimal data they need directly from within their components:

```javascript
// Note.js - Server Component

async function Note(props) {
  // NOTE: loads *during* render, w low-latency data access on the server
  const note = await db.notes.get(props.id);
  if (note == null) {
    // handle missing note
  }
  return (/* render note here... */);
}
```

Waterfalls are still not ideal on the server, so we will provide an API to preload data requests as an optimization. However, since client-server waterfalls are particularly bad for performance, we find not having to worry about them an important benefit.

### Avoiding the Abstraction Tax

React’s use of JavaScript rather than a templating language allows developers to use language features such as function composition and reflection to create powerful UI abstractions. However, when overused these abstractions can come at a cost of more code and more runtime overhead. UI frameworks in “static“ languages can exploit ahead-of-time compilation to compile away some of these abstractions, but this option isn’t available in JavaScript. 

To address this challenge we initially experimented with one approach to ahead-of-time (AOT) optimization — Prepack — but ultimately that direction did not pan out. Specifically, we realized that many AOT optimizations don’t work because they either don’t have enough global knowledge or they have too little. For example, a component might be static in practice because it always receive a constant string from its parent, but the compiler can’t see that far and thinks the component is dynamic. Even when we could make optimizations work, we found that they were unpredictable to the developer. Without predictability it was hard for developers to rely on them.

Server components help to address this by removing the abstraction cost on the server. For example, if a Server Component with multiple layers of wrappers for configurability ends up rendering to a single element, then that's all that will be sent to the client. In the next example, `Note` uses an intermediate `NoteWithMarkdown` abstraction that renders down to a wrapping `<div>`, so React would send just the div (and its contents) to the client:

```javascript
// Note.js
// ...imports...

async function Note({id}) {
  const note = await db.notes.get(id);
  return <NoteWithMarkdown note={note} />;
}

// NoteWithMarkdown.js
// ...imports...

function NoteWithMarkdown({note}) {
  const html = sanitizeHtml(marked(note.text));
  return <div ... />;
}

// client sees:
<div>
  <!-- markdown output here -->
</div>
```

### Distinct Challenges, Unified Solution

As noted above, a theme of these challenges was that React was primarily client-centric. Using traditional server rendering — PHP, Rails, etc — is certainly one way to address some of these challenges, but that approach makes it much harder to build rich, interactive experiences. This reflects a long-standing tension in the world of application development that predates the web: whether to use “thin” or “thick” clients. 

Ultimately, we realized that neither pure server rendering or pure client rendering were sufficient. Server rendering allows applications to easily access server-side data sources and to quickly show static content, while client rendering is critical for rich, interactive features where users expect immediate feedback. But mixing server and client rendering often means mixing *technologies*: writing code in two languages, using two frameworks, keeping two sets of idioms and ecosystems in mind. It also means dealing with cross-language data transfer and often requires duplicating logic, once for server-rendered views and once for interactive client previews.

Server components allow React apps to get the best of both server and client rendering while using a single language, a single framework, and a cohesive set of APIs and idioms. We are still exploring the full design space of Server Components, but we have shipped our first production integration and are actively exploring some additional open areas of research. The high-level design of Server Components is described below.

# Detailed Design

> ⚠️ **Note**: Most React developers do **not** need to understand all the details described here. This section is primarily intended for library or framework authors.

The above sections describe **Server Components** from a developer’s perspective. This section provides more details about the design of Server Components and how they can be integrated into apps and frameworks. First we describe the (simplified) rendering lifecycle of the above notes example. Next we discuss important aspects of the design in further detail. Finally, we close with an overview of some open areas of research that we are actively exploring.

### Simplified Loading Sequence

In this section we’ll review the stages to loading the Note example above, with a root Server Component and child Client Component. Note that some of these stages involve integration with routing and bundling. We expect that initially developers will use Server Components via a framework such as Next.js which would handle this integration by default. These first framework integrations can serve as an example for other libraries and for developers looking to integrate support for Server Components into their apps. 

The process for rendering the Note example is roughly as follows:

* **On the Server:**
    * [framework] The framework’s router will match the requested URL to a Server Component, passing the route parameters as props to the component. It will ask React to render the component and its props: in this case, `Note.js`.
    * [React] React will render the root Server Component and any children that are also Server Components. Rendering stops at native components (divs, spans, etc) and at Client Components. Native components are streamed as a JSON description of the UI, and Client Components are streamed including the serialized props plus a bundle reference to the code for the component. 
        * Note that if any Server Component suspends, React will pause rendering of that subtree and stream a placeholder value instead. When the component is able to continue (un-suspends), React will re-render the component and stream the actual result of the component to the client. You can think of the data streamed to the target as JSON but with slots for components that suspend, where the values to place into those slots are provided as additional items later in the response stream. 
    * [framework] The framework is responsible for progressively streaming the rendered output to the client as React renders each unit of UI.
        * Note that by default React returns a description of the rendered UI, not HTML. This is necessary to allow reconciling (merging) newly fetched data with existing Client Components. A framework may choose to combine Server Components with “server-side rendering” (SSR) to *also* stream the initial render as HTML, which would speed up the initial, non-interactive display of a page.
* **On the Client**
    * [framework] On the client, the framework receives the streamed React response and renders it on the page with React.
    * [React] React deserializes the response and renders the native elements and Client Components. This happens *progressively* — React doesn’t need to wait for the whole stream to finish in order to show something. Suspense allows developers to show intentional loading states while the code for Client Components is loading and while the Server Components are fetching the remaining data.
    * [React] Once all Client Components and all the Server Components output has been loaded, the final UI state is shown to the user. By that point, all Suspense boundaries have already been revealed.

### Update (Refetch) Sequence

Server Components also support reloading to see fresh data. Note that developers don’t fetch individual components in isolation: the idea is that entire subtrees are refetched given some starting Server Component and props. As with initial loading, this would typically involve integration with routing and bundling:

* **On the Client:**
    * [App] The application requests that a given unit of UI be refetched, for example a full route.
    * [framework] The framework orchestrates requesting the rendered result from an appropriate endpoint.
* **On the Server:**
    * [framework] A framework endpoint receives the request and matches it to the requested Server Component. It asks React to render the component and props and handles the stream of rendered results.
    * [React] React renders the components to the target as with initial loading.
    * [framework] The framework handles progressively returning the streamed response data to the client.
* **On the Client:**
    * [framework] The framework receives the streamed response and triggers a re-render of the route using the new rendered output.
    * [React] React reconciles (merges) the new rendered output with the existing components on screen. Because the description of the UI is data, not HTML, React can merge new props into existing components, preserving important UI state such as focus or typing input, or triggering CSS transitions on existing content. *This is a key reason that Server Components return rendered UI output as data (“virtual DOM”) instead of HTML*. 

Key aspects of this process are described in detail below.

### Capabilities & Constraints of Server and Client Components

> ⚠️ NOTE: This section may feel intimidating, but you don’t need to memorize all of these rules to use Server Components. React will provide clear lint, build, and runtime errors for any violations. While the list of the rules appears long, the intuition is simple: Client Components can’t access server-only features like the filesystem, Server Components can’t access client-only features like state, and Client Components may only import other Client Components.

The main new concept introduced in this proposal is **Server Components**. In contrast, **Client Components** are the standard React components that developers are already familiar with: the name “Client Component” doesn’t mean anything new, it’s purely to distinguish them from Server Components. In this section we discuss some important differences between the capabilities of these two types of components. 

* **Server Components:** As a general rule, Server Components run *once per request* on the *server*, so they don’t have state and can’t use features that only exist on the client. Specifically, Server Components:
    * ❌ *May not* use state, because they execute (conceptually) only once per request, on the server. So `useState()` and `useReducer()` are not supported. 
    * ❌ *May not* use rendering lifecycle (effects). So `useEffect()` and `useLayoutEffect()` are not supported.
    * ❌ *May not* use browser-only APIs such as the DOM (unless you polyfill them on the server).
    * ❌ *May not* use custom hooks that depend on state or effects, or utility functions that depend on browser-only APIs.
    * ✅ *May* use `async / await` with server-only data sources such as databases, internal (micro)services, filesystems, etc.
    * ✅ *May* render other Server Components, native elements (div, span, etc), or Client Components.
    * **Server Hooks/Utilities:** Developers may also create custom hooks or utility libraries that are designed for the server. All of the rules for Server Components apply. For example, one use-case for server hooks is to provide helpers for accessing server-side data sources.
* **Client Components:** These are standard React components, so all the rules you’re used to apply. The main new rules to consider are what they can’t do with respect to Server Components. Client Components:
    * ❌ *May not* import Server Components or call server hooks/utilities, because those only work on the server.
        * However, a Server Component *may* pass another Server Component as a child to a Client Component: `<ClientTabBar><ServerTabContent /></ClientTabBar>`. From the Client Component’s perspective, its child will be an already rendered tree, such as the `ServerTabContent` output. This means that Server and Client components can be nested and interleaved in the tree at any level.
    * ❌ *May not* use server-only data sources.
    * ✅ *May* use state.
    * ✅ *May* use effects.
    * ✅ *May* use browser-only APIs.
    * ✅ *May* use custom hooks and utilities that use state, effects or browser-only APIs.

### Sharing Code Between Server and Client

> ⚠️ NOTE: This section may feel intimidating, but you don’t need to memorize all of these rules to use Server Components. React will provide clear lint, build, and runtime errors for any violations. While the list of the rules appears long, the intuition is simple: Client Components can’t access server-only features like the filesystem, Server Components can’t access client-only features like state, and Client Components may only import other Client Components. 

In addition to pure Server Components and pure Client Components, developers may also create components and hooks that work on *both* the server and the client. This allows logic to be shared across environments so long as the components meet *all* the constraints of *both* Server and Client Components. Therefore, shared components and hooks:

* ❌ *May not* use state.
* ❌ *May not* use rendering lifecycle hooks such as effects.
* ❌ *May not* use browser-only APIs.
* ❌ *May not* use custom hooks or utilities that depend on state, effects, or browser APIs.
* ❌ *May not* use server-side data sources.
* ❌ *May not* render Server Components or use server hooks.
* ✅ *May* be used on the server and client.

Although shared components have the most restrictions, we’ve found that in practice many components already obey these rules and can be used across the server and client without modification. Many components simply transform some props based on some conditions, without using state or loading additional data. This is why shared components are the default.

A canonical example of a shared component is something like a Markdown renderer. If we are loading a route to *view* some content written in Markdown (but not edit it), it is more efficient to render the Markdown on the server and avoid downloading the potentially-large Markdown rendering library to the client. But if the user then wants to edit the content, the Markdown library can be downloaded on demand to provide a live preview while the user edits. Having a shared component allows an application to have a single source of truth for this rendering.

### Conventions for Server/Client Components

To mark a component as a Client Component, add `'use client'` directive at the top of your file. This lets you use features like state and event handlers:

```js
'use client';

import { useState } from 'react';

function Button({ children, onClick }) {
  // ...
}
```

You only need to add this directive to the Client Components that you import from Server Components. If you don't add the directive, by default all components will be treated as Shared — imports from the Server are treated as Server Components, and imports from the Client are treated as Client Components. In practice, this means that you only need to annotate Server → Client entry points.

See the [module conventions RFC](https://github.com/reactjs/rfcs/pull/227) for more details.

### Open Areas of Research

Although we have figured out many of the important fundamentals of Server Components and are starting to experiment with them in production, there are still several areas that we are continuing to research. We will continue to share information as we flush out these and other areas:

* **Developer Tools**. We consider first-class developer tools to be a prerequisite for officially releasing any React feature, including Server Components. Our goal is to provide an integrated developer experience for writing React apps that span the server and client. For example, the component inspector might show the Server Component hierarchy, even though those components don’t actually exist in the client tree. Similarly, developers would ideally be able to see server errors and logs show up on the client developer console and be able to breakpoint server code from the client source pane. We’re actively exploring some approaches to realize these ideas in an ergonomic and secure fashion.
* **Routing**. As noted above, applications will generally need some integration with a router in order to map requests for pages into requests to render specific Server Components, and to transmit their rendered output to the client for final display. For the initial release we plan to integrate Server Components into one or more frameworks, which can then serve as a guide for how others can integrate routing. We’ll explore open questions around how routing can work as part of these first framework integrations.
* **Bundling**. Server Components must be able to dynamically send Client Components to the client, including any metadata necessary for the client to fetch the appropriate bundle(s) from a static resource server such as a CDN. As noted in the [Conventions for Server/Client Components](#conventions-for-serverclient-components), bundlers also need to understand the `'use client'` directive and other Server Component-specific package.json conventions. This an area we are actively exploring in collaboration with projects such as Webpack and Parcel. This exploration includes research into different heuristics to find the most efficient bundling strategies.
* **Pagination and partial refetches**. As noted above in [Update (Refetch) Sequence](#update-refetch-sequence), the typical method of reloading a page is to reload it in full. However there are some cases where this is not desirable, such as during pagination. Ideally our app would only fetch the next N items instead of refetching all items that the user has seen up to that point. We are still investigating how best to model pagination with Server Components. For example, internally we are loading Server Components via GraphQL, and using our existing infrastructure for pagination in GraphQL to work around lack of pagination in Server Components. However, we are committed to developing a general solution within React for this use-case.
* **Mutations and invalidations**. In our initial demo we simply clear all cached Server Components whenever an update occurs. This is a simple approach that works well for many apps, but some apps may require more sophisticated invalidation strategies. We are still investigating how to support a generic mechanism for more fine-grained cache invalidation as well as an approach for supporting “optimistic” mutations.
* **Pre-rendering.** One approach for improving page transition times is to pre-render content that the user is likely to interact with. For example, if a user hovers over a link, the application may begin to eagerly load that page. We are still investigating how to support pre-rendering out of the box for applications using Server Components.
* **Static Site Generation**. As with Client Components, Server Components can be rendered at runtime (on the server), build time (locally or on the server), or even some hybrid approach. We are still investigating how to best integrate Server Components and static site generation. For example, the Server Component output stream could be converted into an HTML stream, allowing “static” sites to still benefit from explicitly designed loading states. 

# Drawbacks

* Introducing a new form of components means more to learn.
* The constraints of Server Components may not be intuitive to all users, especially those used to working primarily in only one environment.
* This approach requires a convention for distinguishing server, client, and “shared” components, which may be confusing or off-putting.

# Alternatives

We have investigated various alternatives to this proposal: 

* Client-only rendering
* Server-only rendering
* “Server-side rendering” (SSR)
* Static Site Generation
* AOT compilation

Each of these approaches delivers *some* of the benefits of this proposal, and we expect that *many applications will still benefit from incorporating these approaches*. However, none of these techniques alone is sufficient to achieve the combination of familiar developer experience, user experience, and features achieved by this proposal. 

# Adoption Strategy

As noted above, using Server Components will require integration with an application’s routing system and bundler. To help the community understand how this can work we will initially focus on adding support for Server Components into one or more frameworks. This will make it easy for developers to quickly try out Server Components. It will also serve as a guide for how library authors can add support for Server Components in their libraries and how individual app developers can add support in their apps. 

# How We Teach This

First, note that Server Components are an experimental feature and are not yet at a point where applications can easily adopt them. When we are ready to release Server Components as a stable feature we will update the React documentation, provide examples, and release lint rules that help developers follow the new constraints described above. We’ll also work with one or more frameworks so that developers have an easy way to experiment with Server Components.

# Credits and Prior Art

Server components synthesize ideas from multiple sources:

* Traditional server rendering (ie with PHP, Rails, etc). Specifically, Facebook’s older web architecture which allowed Server Components written in PHP to render native DOM elements or Client Components (w React).
* Relay’s data-driven dependencies, which allow the server to dynamically choose which Client Component to use.
* GraphQL, for demonstrating one approach to avoiding multiple round trips when loading data for client-side apps.

Sebastian Markbage came up with the initial design for Server Components and developed this idea in collaboration with Andrew Clark, Dan Abramov, and Joe Savona. In addition, Lauren Tan, Juan Tejada, Luna Ruan, Andrey Lunyov, Eric Faust, and Royi Hagigi developed the first production integration of Server Components and provided substantial feedback on the design. We’d also like to thank our external partners, Chrome's Aurora team (especially Gerald Monaco, Shubhie Panicker, Nicole Sullivan, and Kara Erickson) and the Vercel team (especially Shu Ding, Jiachi Liu, Tobias Koppers, Javi Velasco, and Tim Neutkens), for their valuable feedback. Additionally, we'd like to thank Fran Dios, Josh Larson, Romuald Brillout, Ward Peeters, and others for their feedback on the Server/Client conventions.

# FAQ

### **Does this replace SSR?** 

No, they’re complementary. SSR is primarily a technique to quickly display a non-interactive version of *client* components. You still need to pay the cost of downloading, parsing, and executing those Client Components after the initial HTML is loaded. 

You can combine Server Components and SSR, where Server Components render first, with Client Components rendering into HTML for fast non-interactive display while they are hydrated. When combined in this way you still get fast startup, but you also dramatically reduce the amount of JS that needs to be downloaded on the client.

### **Does this replace GraphQL?** 

No. GraphQL is one approach for building API endpoints that supports type-safe queries across language boundaries and that can help apps reduce under/over-fetching and reduce round trips (among other features). In contrast, Server Components are focused on building user interfaces.

### Does this replace Apollo, Relay, or other GraphQL clients for React?

No. As noted in the previous question, GraphQL is designed for building API endpoints. As such, GraphQL will continue to be one of many great approaches to loading data for Client Components. In addition, developers may find it helpful to use GraphQL to load data for their Server Components too. For example, internally we use Relay and GraphQL in conjunction with Server Components.

### Is this just working around the lack of a compiler in JavaScript? 

No, a static compiler can help with some things but we found that many real world apps have a lot of dynamic branches such as user-specific options, A/B tests, feature flags, etc. that make static optimizations hit a limit. See [Avoiding the Abstraction Tax](#avoiding-the-abstraction-tax) for more information.

### I cache my stuff in a Service Worker, do I still need this? 

As always, if your existing solution is working well for you then there's no reason to change to something else. See [Motivation](#motivation) and decide whether any of those challenges apply to you!

### Do I have to use this?

If you have an existing client-side app, you can think of all of it as a Client Component tree. If that works for you, great! Server Components extend React to support other scenarios and are not a replacement for Client Components.

### Is this only relevant if my app is Facebook scale? 

No. As noted in the [Motivation](#motivation) section, Server Components were designed to address a number of challenges that React developers face - from folks writing their very first app all the way up to Facebook-scale, and everyone in between. If you’re writing a new app you can directly access the filesystem or a database without standing up an API server. If you already have an app, you can more flexibly access all of your data without having to expose it in your API. 

### Are you reinventing PHP/JSP/whatever for the front end? 

The history of application development is a series of pendulum swings between “thin” and "thick" clients. But the reality is that some parts of any app are more "static" - non-interactive, don’t need immediate data-consistency - while others are "dynamic" and need interactivity and immediate responsiveness. Neither pure server rendering or pure client rendering are ideal for all situations. Server Components — when combined with existing Client Components — allow developers to write each part of their app using the approach that makes the most sense, while sharing a single language and framework and even sharing code across server and client.

### Can I use this in my app today? 

No. See [Adoption Strategy](#adoption-strategy) for more about the status and how this will be rolled out.

### Is this in production at Facebook?

We are running an experiment with a small number of users on a single page, with encouraging results (already ~30% product code size reduction). We expect more savings in an app designed with Server Components for the start.

### Is this specific to Next.js?

No. We *are* partnering with Next.js to build out an initial integration. However, Server Components are designed from the beginning to be used with any framework or integrated into custom application setups. Given the scope of Server Components — including router, bundler, and server/client coordination aspects — we felt that a high-quality initial integration would allow us to demonstrate the setup to other developers.

### What is the adoption story?

See the [Adoption Strategy](#adoption-strategy).

### Why don’t you just use HTML instead of a custom protocol? 

We do want to use streaming HTML for the initial render, but the custom protocol lets us transfer data (component props), and reconcile trees so that the client state as well as DOM focus/scroll/state doesn’t get blown away.

### Isn’t static generation better? 

Static generation is great for some use cases, and you can run Server Components at build time for that.

### What is the debugging story? 

Initially, it’s the same as you would debug your API in Node, but we’re investigating the ability to put Server Components in a worker for DEV so you can debug them side-by-side with client ones. See [Open Areas of Research](#open-areas-of-research) for more thoughts.

### How does composition of server and Client Components work? Are there any limitations?

See [Capabilities & Constraints of Server and Client Components](#capabilities--constraints-of-server-and-client-components) and [Sharing Code Between Server and Client](#sharing-code-between-server-and-client).

### What does it mean for React Native?

We’ve explored an early proof of concept for React Native, but are not currently prioritizing it. In the long term Server Components could be supported by React Native and other renderers.

### Do I have to read my database directly from components?

No, although it might be convenient when you’re starting up. The goal is to be able to scale up or down, and let you move freely between direct database access, microservices, or other approaches *without* rewriting your app. You can also build JavaScript abstractions that let you manage batching queries and provide authorization layers.

### Can I write my own data layer for Server Components?

The API will be documented when it’s stable, but essentially you need to tell React how to deduplicate and cache data requests for your underlying library.

### What is the response format?

It’s like JSON, but with “slots” that can be filled in later. This lets us stream content in stages, breadth-first. Suspense boundaries mark intentionally designed visual state so we can start showing the result before all of it has fully streamed in. This protocol is a richer form that can *also* be converted to an HTML stream in order to speed up the initial, non-interactive render.

### Does this work with TypeScript?

Yes. One limitation is that we cannot currently enforce that the Server/Client boundary is serializable (ie that all props passed to clients are serializable). 

### How does this relate to Suspense?

The Server Components data fetching APIs are integrated with Suspense. We use Suspense to provide loading states and unblock parts of the stream so that the client can show something before the whole response is done.

### How does this relate to Concurrent Mode?

Concurrent Mode is a set of optimizations across React and also integrates with Server Components. For example, Concurrent Mode lets us start rendering Client Components as their data streams in, without waiting for the whole response to be finished.

### How do you do routing?

We don’t know yet. It’s an active area of research. There is a missing feature similar to Context for Server Context that we need to build into React first. 

### Are Server Components refetched whenever their props change?

No, Server Components in the middle of the tree won’t refetch when a parent Client component re-renders. From the Client’s perspective, there are no Server Components at all — it only sees the already resolved trees including divs and other Client components. When a Server component needs to update, such as a result of a mutation, React will refetch the whole Server Component tree output that renders from top to bottom (though finer-grained invalidation is an active area of research). Currently, the Server Component tree starts at the root of the app, but we plan to allow more granular refetching from explicitly chosen entry points.

### Are you always refetching the whole app? Isn’t that slow?

In the demo, we refetch the whole app. This sounds like a lot, but consider that it’s similar to fetching an old-school HTML page. We are planning to introduce a more granular refetching mechanism so that you have the option to refetch only a part of the screen, but it is not available yet.

### What are the performance benefits of Server Components?

Server Components let you put most of your data fetching on the server so that the client doesn’t need to make many requests. This also avoids client network waterfalls, which is typical for fetching in useEffect. Note that this achieves some of the benefits of GraphQL - though of course you may still combine Server Components with GraphQL.

Server Components also let you add non-interactive features to your app without increasing the bundle size. Moving features from the client to the server decreases the initial code size and client JS parse time. Having fewer Client component layers also improves the client CPU time. The client can skip server-generated parts of the tree during reconciliation because it knows they could not have possibly been affected by any state updates.

### Doesn’t always re-fetching the UI make interactions slow?

You can (and should) use Client components at any level of the tree for fast interactions. Also, you don’t want to always refetch — the fetched trees can stay in the client cache and be reused for navigations like back button.

### How is this different from PHP/Rails/etc?

You can reuse some components between Client and Server for use cases with different tradeoffs. E.g. Markdown renderer that offers live preview on the client but is rendered for consumption on the server. This is possible because they’re written in the same paradigm and language.

### How is this different from ASP .NET WebForms?

We don’t send the whole state of the program to the server. Interactive bits should be Client components.

### How is this different from Phoenix LiveView?

Our server isn’t stateful. The tradeoff is we refetch more coarsely.

### What are the downsides of Server Components?

See [Drawbacks](#drawbacks)


### Why not Rx?

Among other things, Rx is a great solution for managing streams of data. We expect that framework authors may find that Rx is a good fit for implementing a transport layer to receive the stream of rendered UI chunks that React creates on the server and stream them to the client.

### What about XSS?

The streaming protocol used to encode Server Component output encodes user-provided input to prevent XSS attacks.

### Happy Holidays!

That's not a question, but happy holidays to you too!
