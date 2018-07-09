- Start Date: 2018-07-09
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Per ["Options for Hardening React &| JSX"][design] this adds a hook API
to allow trusted framework code to prevent XSS by vetting values from
*AttributeExpression*s inside JSX before they reach browser internals.

[Proof of concept code][POC]


# Motivation

```jsx
// Value controlled by attacker
const payload = 'javascript:alert("evil")`

ReactDOM.render(<a href={payload}>...</a>, container)
```

currently creates an `<a>` element that runs `alert("evil")` when clicked.

# Basic example

Framework code should be able to intercept values that might
attacker-controlled before they move from user code to browser
internals.

The proposed API allows a framework to do

```js
const schemeWhitelist = new Set([ 'http', 'https', 'mailto' ]);

ReactDOM.safeInContext = function (node, name, value) {
  // Very simple demo that addresses just the example above.
  if (name === 'href') {  // etc.
    let svalue = `${value}`;
    svalue = svalue.replace(/^[\t\x0a\x0c\x0d]+/, '');  // Strip space
    const match = /^[^/:#?]+:/.exec(svalue);  // Per Std 66 Appendix B
    if (match) {  // Not absolute
      if (!schemeWhitelist.has(scheme.toLowerCase())) {
        return 'about:invalid#unsafeValueRejected';
      }
    }
    return svalue;
  }
  return value;
};
```

Actually writing such a hook is outside the scope of this document but the
author has deployed similar systems that handle nuances related to foreign content,
custom elements, and text nodes in embedded languages.
For example: https://github.com/Polymer/polymer-resin


# Detailed design

["Options for Hardening React &| JSX"][design] discusses this in general terms.

This proposal defines a "safe in context" hook:

```ts
/**
 * Called before React uses a value in a non-DOM-structure altering way.
 *
 * This function should not manipulate the DOM or have side effects visible
 * outside the scope of its call except where necessary to log policy violations.
 *
 * It is the implementor's responsibility to ensure that repeated calls in
 * a tight loop do not deny service, including bounding memory and network
 * usage for frequent policy violations.
 *
 * @param node A node such in the scope of `window.customElements`.
 *    I.e. any implicit adoption ( https://www.w3.org/TR/dom/#dom-core ) should
 *    happen before calling this function.
 *    If value specifies, for example, the text content of a text node under a
 *    `<script>` element then node should be that `<script>` element.
 * @param attributeOrPropertyName The name of the HTML attribute or JS property.
 *    Null iff value specifies textContent/nodeValue for a character data node.
 * @param value the value before any coercion to DOMString so that safeInContext
 *    may treat values as privileged based on their runtime type.
 *
 * @return value or a suitable alternative.  Should not throw.
 */
type SafeInContext = (
  node: Node,
  attributeOrPropertyName: string,
  value: any,
) => any;
```

It adds a property, `ReactDOM.safeInContext` to allow framework code
to specify this hook.

Then it propses to tweak the internals of `ReactDOM.render` to do two things
differently when the hook is present:

*   Before every `setAttribute` or property assignment that results from
    an initial render or update of an element's property map, call
    `ReactDOM.safeInContext` with a *ThisValue* of `null` passing
    as arguments:

    1.  The element
    2.  The attribute or property name.
    3.  The value.

    Use the result of the call as the value to set.
*   Before

    * setting the `nodeValue` or `textContent` of an existing text node
    * or creating a text node to attach to an element or document fragment,

    as a result of initally rendering or updating the children of a DOM node,
    call `ReactDom.safeInContext` with a *ThisValue* of `null` passing
    as arguments:

    1.  The element to contain the text node
    2.  `null`
    3.  The text content.

    Use the result of the call as the text content.

# Drawbacks

## Call overhead

When there is a hook, this adds function call overhead for every attribute
with a non-null value.  The expected hook implementation uses lookup tables
so is lightweight, but adds some overhead too.
The overhead happens during updates so can affect apparent UI latency.

## User space solution

This can be done in user code.  For example, monkeypatching
`ReactDOM.render` to walk its argument.

Hooking into DOMPropertyOperations avoids extra calls to the hook --
only updated properties need vetting.  This approach mitigates much of
the overhead from incremental updates.

## Learnability

This should not complicate teaching React.  The main developer facing
change is that rejected value.  Using a value like `#unsafeValueRejected`
give developers a token that, entered into a search engine, point devs
to relevant framework docs or SO answers.

## Invariant Relaxation

This proposal benefits from [contract values](https://github.com/WICG/trusted-types).

> As described in Christoph Kern's "Securing the Tangled Web", Google
> has been fairly successful at combating DOM-based XSS attacks by
> relying on a set of typed objects instead of strings to represent
> HTML snippets, URLs, etc.

```jsx
const contractValue = /* Object that represents a trusted URL */;

const el = <a href={contractValue}>...</a>;
```

Constructing an element with an object value for an attribute or
text node is currently an invariant violation.

To fully realize this goal, we would need to relax the invariant
violation to allow contract values.

# Alternatives

## Static Analysis

Not fixing this leaves applications vulnerable to XSS via `javascript:` URLs
which are a commonly exploited vector.

Static analysis over JSX expressions might find some problems.  Static
analysis does not mesh well with custom elements or when there are
attribute spread expressions.

## Post processing createElement output

We could monkeypatch `React.createElement` in user code to dynamically
filter ASTs.

This is also bad for incremental updates.

It also lacks context.  Such filters would have to make assumptions that
an AST specifies a DOM tree instead of, for example, React.native or
a network message request body.


## Trusted Types

As alluded to above, the [trusted types][] proposal solves this by checking
values as they cross the IDL bridge from user space to browser builtins,
instead of as they pass from React to browser builtins.

Using trusted types does not prevent using a react value hook or vice
versa, the trusted types polyfill is not without its own overhead,
standards processes iterate much more slowly than frameworks and
library code, and it would be nice to have a consistent way of dealing
with untrusted inputs throughout React{DOM,Native,etc.}.

## DOM mutation events and tree post-processing

Monkeypatching `ReactDOM.render(ast, container, callback)` to
[observe](https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver)
DOM mutation events under container.

<details>
  <summary>Strawman</summary>

```js
let img = document.createElement('img');
let obs = new MutationObserver(
    (events) => {
      for (const event of events) {
        if (event.attributeName === 'onerror') {
          console.log(`cancelled ${event.target.onerror}`);
          event.target.onerror = null;
        }
      }
    });
obs.observe(img, { attributes: true });
img.setAttribute('onerror', 'console.log("ATTACK")');
img.setAttribute('src', 'notfound.gif');
```

</details>

This moves enforcement outside the time slice where `render` happens
so is not obviously in-time to prevent all side effects specified in
payloads.

This would not affect assignment to properties that do not reflect
HTML attributes (e.g. `innerHTML`).

There's also no history of similar approaches protecting deployed
systems.



# Adoption strategy

The proposed hooks provide an opt-in mechanism, so this should not affect
the installed base of React applications.

Turning on hooks that block `javascript:` URLs or other kinds of attribute
values may break existing applications that depend on them.

Hook implementors can mitigate this via two complementary strategies:
1.  https://github.com/babel/babel/pull/8169 proposes an optional change
    to the JSX desugaring that allows distinguishing between
    ```jsx
    (<a href="javascript:ok()">...</a>)
    ```
    and
    ```jsx
    (<a href={payload}>...</a>)
    ```
2.  A hook could operate in a [report only mode][] to ease transition.

    > allows web developers to experiment with policies by monitoring (but not enforcing) their effects



# How we teach this

In-depth documentation and advocacy would be the responsibility of NPM
packages that provide hook implementations and frameworks that turn
them on by default.

This is a "membrane approach."  A membrane sits between two different
sets of objects.  This provides a membrane that separates JavaScript
expressions in JSX from the browser internals.

A product team can define a "policy" that lets them define "safe" per
their project requirements or use a stock policy published on NPM.

Security track talks at React and JS conferences might also be a good
forum to educate developers about handling XSS.

## Changes to React Documentation

https://reactjs.org/docs/introducing-jsx.html#jsx-prevents-injection-attacks
should probably be rewritten, possibly to include an FYI pointing to
popular hook implementations.


# Unresolved questions

*  How to [relax the invariant](#invariant-relaxation) that React elements
   cannot have non-element object values as children or attribute values to
   allow contract values that embody known-safe strings of content.


[design]: https://gist.github.com/mikesamuel/094d3fe4b1b2f702f543fa0c263ca8ff
[trusted types]: https://github.com/WICG/trusted-types
[report only mode]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy-Report-Only
[POC]: https://github.com/mikesamuel/react/commit/a63858f73a379aa1c12af6642ac84bcc8179f69c
