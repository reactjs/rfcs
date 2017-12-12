- Start Date: 2017-12-11
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Add a new, server-side only, static lifecycle method to be invoked after a component and its children have been rendered.

This RFC relates to the proposal to deprecate `componentWillMount` ([RFC #6](https://github.com/reactjs/rfcs/pull/6)) and has been previously discussed on GitHub (see [facebook/react/issues/7671](https://github.com/facebook/react/issues/7671) and [facebook/react/issues/7678](https://github.com/facebook/react/issues/7678)).

# Basic example

Because `componentWillMount` is the only lifecycle method currently invoked during server rendering, it is sometimes used for things like logging, eg:
```js
class Example extends React.Component {
  componentWillMount() {
    if (typeof window === 'undefined') {
      // Server-side
      // Children have not yet rendered
    }
  }
}
```

This RFC proposes to add a new static lifecycle hook named `instanceMounted` to be called after initial server render, eg:
```js
class Example extends React.Component {
  static instanceMounted(props) {
    // Server-side
    // Component and its children have rendered (but not mounted)
    // You can eg save analytics data here
  }
}
```

# Motivation

Provide a place to safely do server-side analytics logging during the "commit" phase.

Historically, `componentWillMount` has been used for server-side functionality, but [RFC #6](https://github.com/reactjs/rfcs/pull/6) proposes to remove that method because it's unsafe for side effects (for reasons explained in the RFC).

## Logging example

This example shows how you might use the new `instanceMounted` method to do log analytics data for server renders.

```js
import ReactDOMServer from 'react-dom/server';
import SomeAnalyticsAbstraction from './path/to/your/code';

const analyticsAPI = new SomeAnalyticsAbstraction();

// This component saves all registered children after rendering
class AnalyticsLogger extends React.Component {
  static instanceMounted(props) {
    analyticsAPI.saveRegisteredComponents();
  };

  render() {
    return this.props.children;
  }
}

// This component self-registers with the analytics API after rendering
class ChildComponent extends React.Component {
  static instanceMounted(props) {
    analyticsAPI.registerComponent(ChildComponent, props);
  };

  render() {
    // Render something meaningful...
  }
}

ReactDOMServer.renderToString(
  <AnalyticsLogger>
    <ChildComponent foo="bar" />
    <ChildComponent foo="baz" />
  </AnalyticsLogger>
);
```

# Detailed design

## `static instanceMounted(props)`

`instanceMounted()` is invoked after a component is server rendered, during the "commit phase". It is a static analog to `componentDidMount()` on the client.

Because this is a static method, it is not possible to call `setState()`. This method should be used only for things like logging analytics data.

# Drawbacks

This proposal is backwards compatible.

# Alternatives

The alternative to this new lifecycle hook is to continue to feature test (eg `typeof window`) inside of either `componentWillMount` or the class constructor, eg:

```js
import ReactDOMServer from 'react-dom/server';
import SomeAnalyticsAbstraction from './path/to/your/code';

const analyticsAPI = new SomeAnalyticsAbstraction();

class ChildComponent extends React.Component {
  componentWillMount() {
    if (typeof window === 'undefined') {
      analyticsAPI.registerComponent(ChildComponent, props);
    }
  };
}

ReactDOMServer.renderToString(
  <Application>
    <ChildComponent foo="bar" />
    <ChildComponent foo="baz" />
  </Application>
);

analyticsAPI.saveRegisteredComponents();
```

# Adoption strategy

Release a minor update to version 16 with support for the new lifecycle hook.

Gather feedback from popular libraries (eg [react-router](https://reacttraining.com/react-router/), [react-intl](https://github.com/yahoo/react-intl)) about this new component method.

# How we teach this

Write a blog post for [reactjs.org](https://reactjs.org/) explaining the motivation for this new hook as well as showing a basic example of its usage. Add it to the [component reference API](https://reactjs.org/docs/react-component.html) as well.

# Unresolved questions

Should we pass `context` to `instanceMounted` as well? This would allow the above example to be rewritten without relying on global variables, eg:

```js
import ReactDOMServer from 'react-dom/server';
import SomeAnalyticsAbstraction from './path/to/your/code';

// This component saves all registered children after rendering
class AnalyticsLogger extends React.Component {
  static childContextTypes = {
    registerComponent: PropTypes.func.isRequired,
  };

  static instanceMounted(props, context) {
    props.analyticsAPI.saveRegisteredComponents();
  };

  getChildContext() {
    return {
      registerComponent: this.props.analyticsAPI.registerComponent,
    };
  }

  render() {
    return this.props.children;
  }
}

// This component self-registers with the analytics API after rendering
class ChildComponent extends React.Component {
  static contextTypes = {
    registerComponent: PropTypes.func.isRequired,
  };

  static instanceMounted(props, context) {
    context.registerComponent(ChildComponent, props);
  };

  render() {
    // Render something meaningful...
  }
}

const analyticsAPI = new SomeAnalyticsAbstraction();

ReactDOMServer.renderToString(
  <AnalyticsLogger analyticsAPI={analyticsAPI}>
    <ChildComponent foo="bar" />
    <ChildComponent foo="baz" />
  </AnalyticsLogger>
);
```

I'm uncertain if this would compatible with [RFC #2](https://github.com/reactjs/rfcs/pull/2) though. At this time I assume it's best _not_ to include a `context` param.