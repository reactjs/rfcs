- Start Date: 2018-04-27
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Introduce a method for unwrapping context value in rendering function.

# Basic example

For example, two contexts are defined in an application:

```javascript
let ThemeContext = React.createContext('light');
let LanguageContext = React.createContext('en-US');
```

Each context represents a single value and a pair of `Provider` and `Consumer`
components to define and get access to this value. Usage details are covered in
[RFC 0002][new-context-rfc].

Current implementation may lead to the code like this:

```javascript
function ThemeAndLanguageConsumer({ children }) {
  return (
    <LanguageConsumer>
      {language => (
        <ThemeConsumer>
          {theme => children({ language, theme })}
        </ThemeConsumer>
      )}
    </LanguageConsumer>
  );
}
```

Suggested API extension allows to shrink the code above to something like this:

```javascript
function ThemeAndLanguageConsumer({ children }) {
  let language = LanguageContext.getValue();
  let theme = ThemeContext.getValue();
  return children({ language, theme });
}
```

# Motivation

While render prop pattern is extremely useful due to the way how it separates
concerns, it affects code complexity by adding nesting levels (basically, each
new context used in a component means +2 levels of indent for its children).
This introduces following concerns:

- Poor readability ([even for React Core Team members][acdlite-tweet])
- Each render method call creates new children functions (since they are inline)

This RFC does not address these concerns for all possible pattern applications.
It reduces related complexity for the components provided by React itself.

# Detailed design

Introduces a new method of `Context` entity:

```javascript
type Context<T> = {
  // existing properties
  Provider: Provider<T>,
  Consumer: Consumer<T>,
  // the suggested method
  getValue: () => T,
};
```

The method can be used in the same places where `Context.Consumer` assumed to
be used.

The upcoming React Suspense feature introduces [effects handling][effects] and
aims at solving async data rendering problems. Despite on being synchronous by
its nature, from the implementation perspective, context unwrapping can be
represented as an effect that is handled in the way that wraps target component
with necessary context consumers so the values become available during
rendering phase. In this case, effects are thrown only once during initial
rendering phase (for mounted component). Subsequent changes of context value
should be covered by the fact the target component wrapped by consumers that
combine their values in a single cache.

# Drawbacks

Since the new method brings yet another way for accessing context value (along
with `Context.Consumer`) there is always a problem to choose the right option.
It usually leads to subjective decisions based on developers experience and
preferences.

Talking about implementation in user space, there is a package that solves
nesting problem by providing composition helpers (using render prop pattern or
higher-order component):
https://www.npmjs.com/package/react-compose-context-consumers

# Alternatives

Alternative design once has been suggested by Andrew Clark in Twitter:
https://twitter.com/acdlite/status/971598256454098944

The idea was to use higher-order components in favor of omitting render props,
which seems to always be an alternative when it comes to comparing these two
patterns in general. However, higher-order components bring its own impact as
a tradeoff.

# Adoption strategy

Described API extension is not a breaking change. The extraction method can be
used by developers in their new components, while being fully compatible with
existing approach that imply using `<Consumer />`.

# How we teach this

Since suggested method has similar background to how React Suspense handles
promises (going from callback style to pull-based code), the same terminology
and flow can be used for describing the concept.

# Unresolved questions

- How React should propagate context value changes to consumers that are using
`getValue()`?
- Given suggested API being implemented, what should be the preferable way of
getting context value?

 [new-context-rfc]: https://github.com/reactjs/rfcs/blob/master/text/0002-new-version-of-context.md
 [effects]: https://github.com/reactjs/react-basic#algebraic-effects
 [acdlite-tweet]: https://twitter.com/acdlite/status/955955121979969537
