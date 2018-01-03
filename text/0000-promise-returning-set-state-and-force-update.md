- Start Date: 2017-01-03
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

This RFC specifies `setState` and `forceUpdate` to return a promise when no
callback is provided.

# Basic example

```js
class DataFetcher extends React.Component {
  state = {
    loading: false,
    error: null,
    data: null,
  };

  async componentDidMount() {
    await this.setState({ loading: true });
    try {
      let response = await fetch(this.props.url);
      let data = await response.json();
      await this.setState({ data, loading: false });
    } catch (error) {
      await this.setState({ error, loading: false });
    }
  };
  
  render() {
    return this.props.render(this.state);
  }
}
```

Type definition:

```js
type Updater<State> = Partial<State> | (prevState: State) => Partial<State>;

declare class React.Component<Props, State> {
  setState(updater: Updater<State>): Promise<void>;
  setState(updater: Updater<State>, callback: () => mixed): void;
  forceUpdate(): Promise<void>;
  forceUpdate(callback: () => mixed): void;
  // ...
}
```

# Motivation

- Promises are JavaScript's primitive for representing async activity
- Async-await is significantly easier to use than callback-based APIs
- Lots of new and existing APIs in the ecosystem and standards are moving towards
  promises (ex: `fetch()`)
- As async-await becomes more available it will make more and more sense build with
- Developers are using `setState` as if it were sync today in lots of places which
  end up needing to be refactored later on when new code needs the state to be applied before running

# Detailed design

When called with a callback, `setState` and `forceUpdate` will continue returning
`undefined`, when called without a callback it will return a promise which resolves
at the same time as the existing callback API.

```js
this.setState(updater, callback); // >> undefined
this.setState(updater); // >> promise
```

The reason for different return values is to avoid messy semantics supporting
the callback and promise simultaneously:

1. Would the callback be called before the promise resolves?
2. Does the promise wait for a promise returned by the callback?
3. What happens when the callback throws synchronously?

Instead, by only supporting promises when a callback is not provided, React
side-steps the problem entirely. There doesn't seem to be a use case where someone
would want both anyways.

# Drawbacks

- If `setState` without the callback is (or could be) optimized in anyway (due to not
  needing the schedule the callback or something), it wouldn't be able to anymore because
  the promise would always have to be returned.
- More API surface area (even if only temporarily)

# Alternatives

Unknown

# Adoption strategy

By supporting both the existing callback API and the promise-returning API, this
feature can be introduced in a minor version and supported indefinitely. If desired
the callback API can eventually start logging warnings and eventually removed in a
major version.

# How we teach this

(I believe) this would be the first place where promises are used within (major?)
React APIs, so teaching promises in the documentation might be necessary. Otherwise,
this API change doesn't change much so it can be taught the same way as it is today.

If React does eventually want to remove the callback API, that can be communicated
through deprecation warnings, blog posts, Dan Abramov tweets, etc.

# Unresolved questions

- Is `setState()` without the callback optimizable in any way that always having to
  always return a promise would prevent?
