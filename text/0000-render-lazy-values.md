- Start Date: 2022-08-26
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

React should accept promises as props and text nodes for native DOM-elements.
While a passed promise has not resolved yet, react should suspend the tree from the DOM-element upwards.
When one such promise resolves, react should trigger a re-render just like it
does with the regular suspense API.

# Basic example

```jsx
const myValue = React.useMemo(() => fetch("/my-api"), [])

return (
    <React.Suspense fallback={<h1 />}>
        <h1>{myValue}</h1>
    </React.Suspense>
)
```

# Motivation

This API would significantly improve the developer experience when building
optimistic UIs. React's real strength compared to other frameworks such as Preact,
Svelte and server-side MVC frameworks such as Phoenix, Laravel and Spring Boot
is that it can handle client side state at scale and therefore support
large application-like interfaces. For application-like interfaces, optimistic UI is
imperative to provide a great user experience.

Passing promises directly to native DOM elements would make it possible to shift
suspending from the component where a fetch call is made towards the actual DOM elements
where the fetched values are needed:
```
old:
<Profile> -> fetches user profile data, suspends
  <ProfileLayout>
    <Avatar />
    <Name />
    <Address />
  </ProfileLayout>
</Profile>

new:
<Profile> -> fetches user profile data
  <ProfileLayout>
    <Avatar /> -> suspends
    <Name /> -> suspends
    <Address /> -> suspends
  </ProfileLayout>
</Profile>
```
This would make it possible to optimistically render a parent component that fetches data
and add suspense boundaries much further down the DOM tree just with react's built-in APIs.

Another motivation for this is to prevent data-fetching waterfalls. With this API, it would
be much easier to render all data-fetching components at once while suspending only the
parts of the tree that actually _use_ the fetched data:

```jsx
// would usually need to suspend to wait until profile is fetched
function App() {
    const profile = React.useMemo(() => fetch("/profile"), [])

    return (
        <>
            <Header profile={profile} />
            <main>
                <OrganizationPage profile={profile} />
            </main>
        </>
    )
}

function Header(props) {
    return (
        <div>
            <nav />
            <React.Suspense fallback={<img src="/avatar-placeholder.svg" />}>
                <img src={props.profile.then(profile => profile.avatarUrl)} />
            </React.Suspense>
        </div>
    )
}

function OrganizationPage(props) {
    const organization = React.useMemo(() => fetch("/organization"), [])

    return (
        <div>
            <React.Suspense fallback={<h1 />}>
                <h1>{organization.then(org => org.name)}</h1>
            </React.Suspense>
            <h3>Members</h3>
            <table>
                <thead>
                    <tr>
                        <th>Name</th>
                        <th>Phone</th>
                    </tr>
                </thead>
                <tbody>
                    <React.Suspense fallback={<TableRowSkeleton />}>
                        {organization.then(org => org.members.map(member => (
                            <tr>
                                <td>{member.name}</td>
                                <td>{member.phone}</td>
                            </tr>
                        )))}
                    </React.Suspense>
                </tbody>
            </table>
        </div>
    )
}
```

This would, of course, also work very nicely with server components, where much more of the DOM could
be streamed to the client before an actual data-fetching call resolves.


# Detailed design

It is really pretty simple from a high-level point of view:
If a native element receives a promise as a prop or child, it suspends.
Once the promise resolves, it renders the resolved value.


# Drawbacks

From my point of view the largest drawbacks are
* it increases the API surface of the react library and this feature might be out of scope.
* it would be a departure from the concept of having native html elements in JSX behave as closely like
  real HTML as possible.
* it introduces additional logic in critical code paths, mainly a check if any value passed to
  a native element is a promise.

# Alternatives

Apollo and other data-fetching libraries already enable optimistic rendering.
Something similar (but far not as elegant) can be implemented as a library and I'm working on that.

# Adoption strategy

This is not a breaking change. Libraries could slowly adopt this strategy instead
of their custom optimisic rendering solutions.

# How we teach this

I suppose this should become the recommended way to use suspense for data fetching and
would expect this to get its own section in the docs.

As gradual adoption is not a problem at all developers would learn about it through the
regular release notes and release channels. Consumers of data fetching libraries and frameworks
would probably learn about it through their library or framework's release notes.

I would e.G. love to have an option to have Next.js render a page immediately as soon as the
js-bundle is loaded and pass server side props as a promise to a page and would expect such features
to show up in Next.js' release notes.

# Unresolved questions
¯\_(ツ)_/¯
