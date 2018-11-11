- Start Date: 2018-11-11
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Introduce a new hook to allow extraction of server-side data and
subsequent client-side hydration along the component tree.

# Basic example

```js
// Usage with react-cache
function ResourceProvider({ children, fetch, hash }) {
    const resource = useSerializable(
        // Create a "resource" object acting as a cache
        // and manager for data requests
        //
        // The `data` argument is optional and may
        // contain data serialized by the server
        data => createResource(fetch, hash, data),
        // Collect the serializable data from the
        // resource object
        resource => resource.getSerializable(),
    );

    // The resource object is returned from the hook function
    // and can be used as a context to fetch data
    return (
        <ResourceContext.Provider value={resource}>
            {children}
        </ResourceContext.Provider>
    );
}

// Usage with Redux
function ReduxProvider({ children, reducer }) {
    const store = useSerializable(
        data => createStore(reducer, data),
        store => store.getState(),
    );

    return (
        <ReduxContext.Provider value={store}>
            {children}
        </ReduxContext.Provider>
    );
}

// Usage with Apollo
function ApolloProvider({ children, config }) {
    const client = useSerializable(
        data => {
            const cache = config.cache || new InMemoryCache();
            if(data) {
                cache.restore(data);
            }

            return new ApolloClient({ ...config, cache });
        },
        client => client.cache.extract(),
    );

    return (
        <ApolloContext.Provider value={client}>
            {children}
        </ApolloContext.Provider>
    );
}
```

# Motivation

While the Suspense API provides a clear way to handle data fetching in
a React tree, there is currently no way to extract and serialize said
data alongside a server-rendered tree (this is probably outside of the
scope of Suspense, and right now Suspense is not supported on the
server anyway).

Currently the most commonly used solution to handle both loading and
serializing server-side data is next.js' `getInitialProps` method. But
this solution relies on HOC, and as such suffers from the same
problems (proliferation of components in the tree, potential prop name
collisions). Finally, it doesn't integrate well with Suspense (since
the data fetching is expected to happen in `getInitialProps` before
rendering), especially with code-splitting and `React.lazy` since
`getInitialProps` is a static method that is called before rendering
the component.

This RFC proposes a completely independent solution from Suspense to
allow declaring a "serializable" data store during a component's
render phase, after it has potentially been lazy-loaded. This would
allow ReactDOMServer to extract this data after rendering the tree,
and ReactDOM to provide it back to the component during the hydration
phase. This essentially hands the concern of handling server-side data
to React, just like the Hooks proposal did with the concern of
handling state and side-effects.

# Detailed design

Introduce a new Hook function with the following signature:

```js
function useSerializable<R, D>(createResource: (data?: D) => R, collectData: (resource: R) => D): R;
```

The `createResource` function is called synchronously when the
component is mounted, and the resulting resource object is stored in
the hook state chain and returned (this part is essentially doing
`useMemo(createResource, [])`).

The `collectData` function is called at the very end of the render
when all the suspended components have resolved, with the resource
object as its argument, and should return a serializable object.

Since as mentionned earlier there is currently no support for Suspense
server-side, how the serialized data are actually collected is not
currently part of this RFC. Tentatively, the following API could be
proposed:

```js
const [html, data] = await renderToStringAsync(<AsyncTree />);
```

Where `html` contains the rendered tree in string form, and `data`
contains an *arbitratry* serializable object, which could then be
provided as an additional argument to the `hydrate` function:

```js
hydrate(<AsyncTree />, container, callback, data);
```

When mounting the tree, React will now call `createResource` with the
corresponding fragment of data. It's important to note that aside from
being serializable (can be passed to `JSON.stringify` without error),
the format of the `data` object is completely unspecified and should
be handled as an opaque data type. This allows the React
implementation to layout the data as it see fit (though the format
should probably remain stable in a minor release, even if different
client- and server-side versions of ReactDOM should be rare).

# Drawbacks

This RFC extends the API surface of React by pushing an additional
concern into the core library. Internally, this may increase the
complexity of both the server-side and client-side renderers in order
to keep track of where the data should be injected.

In theory this API could be implemented in userland, with one major
difference: manual keys. Since an external library cannot make any
assumption about the React rendering process, `useSerializable` needs
an additional `key: number | string` parameter to match server data
with client components, and just like the `getInitialProps` / HOC
approach this is potentially open to name collisions.

Finally, this ia smaller nitpick but I don't see a reliable way to run
dead code elimination on the `collectData` parameter to ensure that
this code gets removed in client-side builds.

# Alternatives

- Keep server data outside of React core (potentially falling back to the aforementioned userland implementation)
- Use another name instead of `useSerializable`
- Static `createContext` and `collectData` methods for class components

# Adoption strategy

While not a breaking change for React itself, this API could become
one for various SSR Frameworks. But due to the way `React.lazy` works
releasing Suspense for the servers will be breaking anyway, and this
API could become a small part of that bigger release.

# How we teach this

Due to the cross-environment communication required to pass the data
emitted by the server renderer back to the client renderer, aside from
adding documentation for the new hooks and the `ReactDOMServer` and
`ReactDOM` methods, a new page in the `Advanced Guides` of the
documentation could be added to cover all the aspects of server-side
rendering, including this one.

Ideally though, for most users the details of the serialization and
hydration would be abstracted away by a framework, leaving the
`useSerializable` hook (or a custom hook based on it) as the only part
of the API effectively exposed to the end user. Since this would be a
core API some libraries may even afford to have the call to
`useSerializable` as part of their Provider component, meaning the end
user gets server data serialization "for free" complexity-wise.

# Unresolved questions

Should a "reverse" API be exposed (`renderToString` with an existing
data payload, and calling `collectData` on a mounted root) ?
This would allow some advanced caching scenarios on both the client
and the server, especially for offline use in conjunction with shared
workers. But just like the actual API to collect or hydrate data this
is probably outside of the scope of this RFC.
