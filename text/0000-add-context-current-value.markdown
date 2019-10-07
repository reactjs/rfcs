- Start Date: 2019-10-07
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Add a `Context.currentValue` method as part of Context's public API.

# Basic example

```javascript
const MyContext = React.createContext({ name: "foo" });

// inside of a class component constructor or lifecycle method
class FooComponent {
  constructor(props) {
    super(props);
    this.name = MyContext.currentValue.name;
  }
}
```

Or a `useQuery`-ish API that is available to class components:

```
class FooComponent {
  constructor(props) {
    super(props);
    // newQuery is able to access the ApolloProvider.currentValue 
    this.query = ReactApollo.newQuery({ ... });
  }

  render() { 
    const { ... } = this.query;
  }
}
```

Or as field initializers:

```
class FooComponent {
  // newQuery is able to access the ApolloProvider.currentValue 
  query = ReactApollo.newQuery({ ... });

  constructor(props) {
    super(props);
  }

  render() { 
    const { ... } = this.query;
  }
}
```

# Motivation

The motivation is to provide class components (_and_ code they might invoke) with a non-HOC,
non-render props, non-`contextType` way to access `Context` values, similar to what hooks
are able to do with `useContext`.

Currently `contextType` allows a very limited form of this, but couples the component to
a single context, and does not allow the component to invoke reusable code that, on behalf
of the class component, can easily access whatever contexts it wants (i.e. the GraphQL
context or the Redux context).

The intent is basically to "level the playing field" between FP components and class components,
by providing class components with what (IMO) is the biggest win of hooks: easy, intuitive,
decoupled access to contexts from utility/infrastructure libraries like react-apollo/etc.

(Well, that and a way to access the component's lifecycle methods in a decoupled manner, but that
is a separate topic/RFC.)

My general assertion is that, because a `MyContext.currentValue` does not currently exist, many
libraries like react-apollo are focusing on hook-only APIs and defacto pushing the React community
towards FP components whether projects really want to use FP components or not, i.e. to use the
latest non-HOC/non-render prop APIs, they _have_ to use hooks.

# Detailed design

I personally don't have enough experience yet to flush this out, but would be happy
to work with someone on it. Basically I'd need a sponsor to help with the implementation.

I've poked around the `useContext` implementation to try and mirror what it does
when called from class components, and have a "technically works" but I know
non-viable approach that just accesses the private `_currentValue` field.

To do this right, `Context.currentValue` would need to have similar lifecycle
rules applied to when it can be called (similar to although different than `useContext`'s
restrictions), i.e. `currentValue` could only be called from places where React knows
the provider's current value due to having actively walked the component tree, i.e.:

* Class component constructors, or
* Class component lifecycle methods, i.e. `render`, `componentDidMount`, etc.

I assume that these restrictions is why `Context.currentValue` does not already exist
today, i.e. the current HOC and render prop approaches avoid the need for "can only be
called $here" enforcement by providing snapshots of the value behind the Consumer
component's API.

However, as far as I understand, hooks have shown/set a precdence for some amount
of runtime API enforcement, i.e. "can only be called $here", and so I think it is
fair game to allow class components access to APIs that rely on that level of
enforcement as well.

# Drawbacks

Why should we *not* do this?

- It prolongs the "class vs. FP" divide by giving class components a lifeline to
  hook-level ease-of-use APIs.

  Given the popularity of hooks-style APIs, the "class vs. FP" dictomy will likely
  very naturally play itself out in favor of FP, which is perhaps the intent of
  hooks / direction of the React core team in the first place.

- Callers would be responsible for "refreshing" the context value, i.e. if they
  capture it in the component constructor, and then the context value changes,
  the `componentDidUpdate` method would fire, and the caller would be responsible
  for "seeing" the new context value.

  So this does expose a potential foot-gun for users to misuse, i.e.:

  ```javascript
  class Component {
    constructor(props) {
      super(props) {
        this.context = MyContext.currentValue;
      }
    }

    componentDidUpdate() {
      // don't forget to check this
      this.context = MyContext.currentValue;
    }
  }
  ```

  My assertion is that ideally this `Context.currentValue` method would be used primarily
  from instrastructure libraries (similar to react-apollo) that would encode the best
  practice of both capturing the initial value + updating it when it changes. (I.e. if
  you look in the current react-apollo hook code, they have a `refreshClient` method that
  does some of this.)

  - One potential mitigation for this would be to name the `Context.currentValue` API with a
    very explicit hint, i.e. `Context.currentValueMustBeRefreshedManually`.

    Although initially ugly, this should not actually hurt actual-component UX because my intention
    is to provide an API for infrastructure libraries to use, and not something that would be
    regularly called directly from components themselves.

  - Another mitigation would be to only return a function that returns the value, i.e.:

    ```javascript
    // in constructor/initialization/etc.
    this.contextFn = MyContext.currentValueFn;

    // on usage
    this.contextFn.get();
    ```

    As this would help prevent capturing the potentially-stale context value (although obviously
    not completely prevent it as users could still call `MyContext.currentValueFn.get()` right
    away.)


# Alternatives

What other designs have been considered? What is the impact of not doing this?

# Adoption strategy

This would not be a breaking change, so adoption should be easy/incremental.

# How we teach this

I believe teaching is easy because `Context.currentValue` seems, to me, like a very
intuitive thing to have, i.e. Context mirrors thread locals in other languages, and
if anything _not_ being able to access the thread locals current value is more
surprising from a teaching method.

That said, it would come with the lifecycle restrictions of "can only be accessed
when we know where in the component tree we are at", but hooks have already shown
that these sorts of constraints can be taught.

# Unresolved questions

