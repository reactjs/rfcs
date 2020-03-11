- Start Date: 2020-02-11
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Provide a mechanism for DevTools (or another browser extension) to switch a
page to a development build of React.

# Motivation

Every once in a while, all of us end up debugging a production build of our
apps. Maybe there's a bug that only shows up in a production app, or maybe
you're troubleshooting an issue a user's experiencing that you can't
reproduce locally.

React recommends developing against a development build for a reason—a lot
of weird React bugs are detected and warned about, but only in development
builds. If there was a quick way to run "the prod build of my site, but
with a development build of React," debugging some of these issues could
be substanially easier.

The specific issue that inspired my to file this RFC was an issue with
Gatsby SSR: Gatsby only uses SSR in production, and we had a bug that
caused SSR to generate a slightly different component tree than the
client, causing a hard-to-track-down issue with hydration. A development
build of React would've detected this issue and given us a helpful warning,
but the bug didn't occur in development.

# Detailed design

We could wrap the production builds of each package (react, react-dom, etc)
in something like this (for a commonjs build; umd would be a little more 
complicated but similar in spirit):

```js
if(window.__injectReactImplementation) {
  window.__injectReactImplementation('react-dom', require, module) // possibly also pass version
} else {
  // The actual production build of the package
}
```

A browser extension could then, when requested by the user, define
`__injectReactImplementation` (name bikesheddable; maybe this fits into
the existing "devtools hook" structure?) to load a development (or
profiling) build of the requested package.

# Drawbacks

- This feels like a fairly "intrustive" change for a relatively obsure
  use case.
- Perhaps some production-only bugs happen *because of* the production
  react build; having this as a common debugging step could lead
  people astray when trying to find a bug like this.
- Might encourage bad habits of developing against a production build
  and only switching to a dev build when necessary for debugging, which
  might make things like missing keys more likely to go unnoticed.
- I'm a little worried that this add to the attack surface of React. At a
  first glance, it's OK—if you can set window.__injectReactImplementation
  to an arbitrary function, you probably have the ability to execute
  arbitrary JS anyway. But it seems vaguely plausible that this could be
  used as part of an exploit chain, so this is worth a look from someone
  with more security knowledge than I have.

# Alternatives

- Find another way to replace the React implemenation, such as via intercepting
  the request for the JS file or by modifying the app's build system to support
  a "partial production" build.
  - Both of these might allow this to be done without any added complexity on
    React's end, but I think my proposal is less complex to implement overall.
- Settle with bad DX in an obscure situation.
  - Maybe this sort of situation (debugging a production build) just doesn't
    come up often enough for it to be worth the complexity required to make
    it easier.

# Adoption strategy

Not a breaking change, nor something we'd need developer to adopt *en masse*.
Someone would at need to write the browser extension that used this—this
could be something that React maintained as part of the Devtools, or just a
hook for third-party extensions to use.

# How we teach this

Don't think it would need to be much more that a blog post announcing the
feature in DevTools, or just a changelog entry if devtools doesn't hook into
this—it's not a major paradigm shift we need to disseminate widely.

# Unresolved questions

I'm not familiar enough with Webpack internals to know if the design
I proposed would work with its static analysis—if it doesn't, making
that work (such as by moving any requires outside of the if and passing
them in to __injectReactImplementation) may add a fair bit of complexity.
