- Start Date: 2017-12-11
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Provide a server-side rendering equivalent for `componentDidMount`.

This RFC relates to the proposal to deprecate `componentWillMount` ([RFC #6](https://github.com/reactjs/rfcs/pull/6)) and has been [previously discussed on GitHub](https://github.com/facebook/react/issues/7671).

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

  componentDidMount() {
    // Client-side
  }
}
```

This RFC proposes to add a new lifecycle hook named `componentDidServerRender` to be called after initial server render, eg:
```js
class Example extends React.Component {
  componentDidServerRender() {
    // Server-side
    // All children have rendered but NOT mounted
    // It is not safe to use refs to interact with eg the DOM
  }

  componentDidMount() {
    // Client-side
    // All children have rendered and been mounted
    // You can now safely use refs to interact with eg the DOM
  }
}
```

# Motivation

Provide a place for initialization logic with side effects to safely run on the server. This includes things like:
* [Logging/metrics](#logging-example)
* [Smarter default server-side behavior for virtualization/windowing libraries](#server-side-fallbacks-for-virtualizationwindowing-libraries)
* [Routing redirects if match was found (eg the legacy react-router `Match`/`Miss` components)](#routing-redirect-example)
* Initializaing shared localization data

Typically, `componentWillMount` is used for this, but [RFC #6](https://github.com/reactjs/rfcs/pull/6) proposes to remove this method. (It is not a very ergonomic solution anyway.) The logic could be moved to the component constructor but it's generally considered bad practice for a constructor to have side effects.

## Logging example

This example shows how you might use the new `componentDidServerRender` method to do log analytics data for server renders.

```js
import API from './your-api-abstraction';

class AnalyticsComponent extends React.Component {
  componentDidServerRender() {
    API.logSomeData();
  }
}
```

## Server-side fallbacks for virtualization/windowing libraries

This example shows how a windowing library (like [react-virtualized](https://github.com/bvaughn/react-virtualized)) might implement a smarter default behavior when being rendered on the server.

Windowing libraries typically require their dimensions to be injected via props in order to function. These measurements typically come from the DOM, after initial mount, and are unavailable on the server. This means that be default, no windowed content will render. A component like [`AutoSizer`](https://github.com/bvaughn/react-virtualized/blob/master/source/AutoSizer/AutoSizer.js) could use this new lifecycle hook to determine that it's been server-rendered and default to showing a fixed amount of content:

```js
class AutoSizer extends React.Component {
  static defaultProps = {
    serverHeight: 400,
    serverWidth: 800,
  };

  state = {
    height: 0,
    width: 0,
  };

  componentDidMount() {
    // Measure actual DOM element size and store in state
  }

  componentDidServerRender() {
    // Use a default size to render *some* content
    this.setState({
      height: this.props.serverHeight,
      width: this.props.serverWidth,
    });
  }

  render() {
    const { height, width } = this.state;

    return this.props.children({
      height,
      width,
    });
  }
}
```

The above example could be re-written to _assume_ a server context by default (with a flag in `state` that is updated by `componentDidMount`). This could result in the windowing component creating more content than is necessary on the initial render though (eg if the server fallback was larger than the actual measured size).

## Routing redirect example

This example shows how you might use the new `componentDidServerRender` method (along with `context`) to create a router that renders a default view if the current URL does not match one of the specified patterns. (Note that this example is over-simplified and a little silly.)

```js
class Router extends React.Component {
  static childContextTypes = {
    hasMatches: PropTypes.func.isRequired,
    registerMatch: PropTypes.func.isRequired,
  };

  static GlobalState = {
    matched: false,
    hasMatches: () => Router.GlobalState.matched,
    registerMatch: matched => {
      if (matched) {
        Router.GlobalState.matched = true;
      }
    },
    resetMatched: () => {
      Router.GlobalState.matched = false;
    },
  };

  getChildContext() {
    return {
      hasMatches: Router.GlobalState.hasMatches,
      registerMatch: Router.GlobalState.registerMatch,
    };
  }

  componentWillUpdate() {
    Router.GlobalState.resetMatched();
  }

  render() {
    return this.props.children;
  }
}

class Match extends React.Component {
  static propTypes = {
    children: PropTypes.node.isRequired,
    location: PropTypes.object.isRequired,
    pattern: PropTypes.string.isRequired,
  };

  static contextTypes = {
    registerMatch: PropTypes.func.isRequired,
  };

  componentDidMount() {
    this._registerMatch();
  }

  componentDidServerRender() {
    this._registerMatch();
  }

  componentDidUpdate() {
    this._registerMatch();
  }

  render() {
    return this._doesMatch()
      ? this.props.children
      : null;
  }

  _doesMatch() {
    const { location, pattern } = this.props;

    // This is over-simplified :)
    return location.pathname === pattern;
  }

  _registerMatch() {
    const { registerMatch } = this.context;
    const { pattern } = this.props;

    registerMatch(this._doesMatch());
  }
}

class Miss extends React.Component {
  static propTypes = {
    children: PropTypes.node.isRequired,
  };

  static contextTypes = {
    hasMatches: PropTypes.func.isRequired,
  };

  state = {
    hasMatchesInContext: true
  };

  componentDidMount() {
    this._checkForMatches();
  }

  componentDidServerRender() {
    this._checkForMatches();
  }

  componentDidUpdate() {
    this._checkForMatches();
  }

  render() {
    return this.state.hasMatchesInContext
      ? null
      : this.props.children;
  }

  _checkForMatches() {
    const { hasMatchesInContext } = this.state;
    const { hasMatches } = this.context;

    if (hasMatches() !== hasMatchesInContext) {
      this.setState({
        hasMatchesInContext: hasMatches(),
      });
    }
  }
}

const App = () => (
  <Router>
    <Match location={location} pattern='/some/url'>
      <SomeView />
    </Match>
    <Match location={location} pattern='/some/other/url'>
      <SomeOtherView />
    </Match>
    <Miss>
      <DefaultView />
    </Miss>
  </Router>
);
```

# Detailed design

## `componentDidServerRender()`

`componentDidServerRender()` is invoked immediately after a component is server rendered. It is the server equivalent to `componentDidMount()`.

Calling `setState()` in this method will trigger an extra rendering, but it is guaranteed to flush during the same tick. This guarantees that even though the `render()` will be called twice in this case, the user won't see the intermediate state.

Use this pattern with caution because it can impact performance. However, it is sometimes necessary for cases like localization or routing where you need to override the results of the initial render based on new information.

# Drawbacks

This proposal is backwards compatible.

# Alternatives

The alternative to this new lifecycle hook is to continue to feature test (eg `typeof window`) inside of either `componentWillMount` or the class constructor.

# Adoption strategy

Release a minor update to version 16 with support for the new lifecycle hook.

Pending the outcome of [RFC #6](https://github.com/reactjs/rfcs/pull/6), include messaging about this new hook in the `componentWillMount` deprecation warning.

Coordinate with popular libraries (eg [react-router](https://reacttraining.com/react-router/), [react-intl](https://github.com/yahoo/react-intl)) to ensure this new hook meets their needs before `componentWillMount` is deprecated. 

# How we teach this

Write a blog post for [reactjs.org](https://reactjs.org/) explaining the motivation for this new hook as well as showing a basic example of its usage. Add it to the [component reference API](https://reactjs.org/docs/react-component.html) as well.

# Unresolved questions

None?