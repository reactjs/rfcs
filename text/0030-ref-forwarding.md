- Start Date: 2018-03-07
- RFC PR: https://github.com/reactjs/rfcs/pull/30
- React Issue: https://github.com/facebook/react/pull/12346

# Summary

Provide an API that enables [`refs`](https://reactjs.org/docs/refs-and-the-dom.html) to be forwarded to a descendant (child or grandchild component).

# Motivation

I recently began contributing to [`react-relay` modern](https://github.com/facebook/relay/tree/master/packages/react-relay/modern) in order to prepare it for the [upcoming React async-rendering feature](https://reactjs.org/blog/2018/03/01/sneak-peek-beyond-react-16.html).

One of the first challenges I encountered was that `react-relay` depends on the legacy, unstable context API, and values passed through legacy `context` aren't accessible in the new, [static `getDerivedStateFromProps` lifecycle](https://github.com/reactjs/rfcs/blob/master/text/0006-static-lifecycle-methods.md#static-getderivedstatefrompropsnextprops-props-prevstate-state-shapestate--null). The long-term plan for `react-relay` is to use the [new and improved context API](https://github.com/reactjs/rfcs/blob/master/text/0002-new-version-of-context.md) but this can't be done without dropping support for older versions of React- (something `react-relay` may not be able to do yet).

In an effort to work around this, I created a smaller wrapper component to convert the necessary `context` value to a `prop` so that it could be accessed within the static lifecycle:

```js
function injectLegacyRelayContext(Component) {
  function LegacyRelayContextConsumer(props, legacyContext) {
    return <Component {...props} relay={legacyContext.relay} />;
  }

  LegacyRelayContextConsumer.contextTypes = {
    relay: RelayPropTypes.Relay
  };

  return LegacyRelayContextConsumer;
}
```

Unfortunately, this change broke several projects within Facebook that depended on refs to access the inner Relay components.

As I considered this, I became convinced that the new Context API naturally lends itself to similar wrapper components, e.g.:
```js
const ThemeContext = React.createContext("light");

function withTheme(ThemedComponent) {
  return function ThemeContextInjector(props) {
    return (
      <ThemeContext.Consumer>
        {value => <ThemedComponent {...props} theme={value} />}
      </ThemeContext.Consumer>
    );
  };
}
```

Unfortunately, within the current limitations of React, there is no (transparent) way for a 
a `ref` to be attached to the above `ThemedComponent`. Common workarounds typically use a special prop (e.g. `componentRef`) like so:
```js
const ThemeContext = React.createContext("light");

function withTheme(ThemedComponent) {
  return function ThemeContextInjector(props) {
    return (
      <ThemeContext.Consumer>
        {value => (
          <ThemedComponent {...props} ref={props.componentRef} theme={value} />
        )}
      </ThemeContext.Consumer>
    );
  };
}
```

This convention varies from project to project, and requires users of the `ThemedComponent` to be aware of the fact that it's wrapped by a higher-order component. This detail should not be important. It should be possible to use a standard React `ref` that the `ThemeContextInjector` (in this case) could forward to its child `ThemedComponent`.

The idea of `ref`-forwarding [is not new](https://github.com/facebook/react/issues/4213), but the new context and other efforts like [`create-subscription`](https://github.com/facebook/react/pull/12325) greatly increase the importance of this feature.

# Basic example

The Relay `context` injector component ([shown above](#motivation)) could use the ref-forwarding API proposed by this RFC as follows:

```js
function injectLegacyRelayContext(Component) {
  function LegacyRelayContextConsumer(props, legacyContext) {
    return (
      <Component
        {...props}
        relay={legacyContext.relay}
        ref={props.forwardedRef}
      />
    );
  }

  LegacyRelayContextConsumer.contextTypes = {
    relay: RelayPropTypes.Relay
  };

  // Create a special React component type that exposes the external 'ref'.
  // This lets us pass it along as a regular prop,
  // And attach it as a regular React 'ref' on the component we choose.
  return React.forwardRef((props, ref) => (
    <LegacyRelayContextConsumer {...props} forwardedRef={ref} />
  ));
}
```

The theme context wrapper could as well:

```js
const ThemeContext = React.createContext("light");

function withTheme(ThemedComponent) {
  function ThemeContextInjector(props) {
    return (
      <ThemeContext.Consumer>
        {value => (
          <ThemedComponent {...props} ref={props.forwardedRef} theme={value} />
        )}
      </ThemeContext.Consumer>
    );
  }

  // Forward refs through to the inner, "themed" component:
  return React.forwardRef((props, ref) => (
    <ThemeContextInjector {...props} forwardedRef={ref} />
  ));
}
```

Here is another example [Dan mentioned on Twitter](https://twitter.com/dan_abramov/status/974008682311815169):

```js
// Tell React I want to be able to put a ref on LikeButton.
const LikeButton = React.forwardRef((props, ref) => (
  <div className="LikeButton">
    {/* Forward LikeButton's ref to the button inside. */}
    <button ref={ref} onClick={props.onClick}>
      Like
    </button>
  </div>
));

// I can put a ref on it as if it was a DOM node or a class!
// (In this example, the ref points directly to the DOM node.)
<LikeButton ref={myRef} />;
```

# Detailed design

Hopefully the usage of this API is clear from the [above examples](#basic-example), so in this section I'll outline a possible implementation strategy.

The `forwardRef` function could return a wrapper object (similar to what the context API uses) with a `$$typeof` indicating that it's a ref-forwarding component. When React encounters this type, it can assign a new type-of-work to React (e.g. `UseRef`). The "begin" phase could then invoke the render prop argument, passing it `workInProgress.pendingProps` and `workInProgress.ref` to create the children, and then continue reconciliation.

# Drawbacks

This API increases the surface area of React slightly, and may complicate compatibility efforts for react-like frameworks (e.g. `preact-compat`). I believe this is worth the benefit of having a standardized, transparent way to forward refs.

# Alternatives

1. Add a new class method, e.g. `getPublicInstance` or `getWrappedInstance` that could be used to get the inner ref. (Some drawbacks listed [here](https://github.com/facebook/react/issues/4213#issuecomment-115019321).)

2. Specify a ["high-level" flag on the component](https://github.com/facebook/react/issues/4213#issuecomment-115048260) that instructs React to forward refs past it. This approach could enable refs to be forwarded one level (to the immediate child) but would not enable forwarding to deeper child, e.g.:

```js
const ThemeContext = React.createContext("light");

function withTheme(ThemedComponent) {
  return function ThemeContextInjector(props) {
    return (
      <ThemeContext.Consumer>
        {value => (
          // ref belongs here
          <ThemedComponent {...props} ref={props.componentRef} theme={value} />
        )}
      </ThemeContext.Consumer>
    );
  };
}
```

3. [Automatically forward refs for stateless functions components](https://github.com/facebook/react/issues/4213#issuecomment-115051991). (React currently warns if you try attaching a `ref` to a functional component, since there is no backing instance to reference.) This approach would not enable class components to forward refs, and so would not be sufficient, since wrapper components often require class lifecycles. It would also have the same child-depth limitations as the above option.

# Adoption strategy

This is a new feature. Since there are no backwards-compatibility concerns, adoption can be organic.

One candidate that would immediately benefit from this feature would be [`create-subscription`](https://github.com/facebook/react/pull/12325).

# How we teach this

Add a section to the ["Refs and the DOM" documentation page](https://reactjs.org/docs/refs-and-the-dom.html) about ref-forwarding.

Write a blog post about the new feature and when/why you might want to use it. Highlight examples using the new context API.

# Unresolved questions

None presently.