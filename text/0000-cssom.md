- Start Date: 2020-04-20
- RFC PR:
- React Issue:

# Summary

I know it's a long-lasting and controversial topic and probably not the highest priority for the team right now, but let me try to create the first **simplified** vision for a CSS implementation we all could agree on.

CSS-in-JS like React is on a dual mission, supporting both complex apps and small(ish) sites. This has created a big divide in the community because of the very different needs that these types of projects have. Like React, CSS-in-JS was born out of the need to reduce the complexity of maintaining large applications.

While there are many options for authoring CSS, it is super tiresome to convince a team to use a particular approach. It creates a big divide and lots of frustration. While there is a huge value in not having a built-in solution and giving the space to 3rd party libs to thrive and experiment, the friction and frustration for not having a good built-in approach that covers 90% of use cases is real.

## Motivation

### Goals

- CSS scoping (unique selector)
- High-performance rendering of static styles via CSSOM.
- CSS Rules can be reused across components.
- Components have no additional overhead for styling.
- Lists - inline styles have no way to be reused across list elements, where typically styles are repeated.
- Support the full CSS spec. Inline styles are limited to [CSSStyleDeclaration](https://developer.mozilla.org/en-US/docs/Web/API/CSSStyleDeclaration) and we need to support the full spec on the web.
- Adhere to React's component model and the upcoming Suspense architecture.

### Non-goals

- Enforcing CSS-in-JS over vanilla CSS or css-modules (you can still use className)
- Implementing every possible idea CSS-in-JS libs came up with

# Detailed design

### Tagged templates

- familiar syntax
- syntax highlighting (already supported)
- linting
- ability to return a data structure that React can identify, not just a string
- interpolations

```js
import { css } from "react-cssom";
const buttonStyle = css`
  color: red;
`; // {css: '.css-0 { color: red; }'}

render(<button style={buttonStyle}>x</button>);
```

### Using style prop for both inline and CSSOM styles

Since we can return from the tag function any data structure we want, we can identify styles created using our `css` function and we have no problem accepting both types of styles.

Using the `style` prop allows us to be flexible in the future about how React wants to optimize and treat this interface.

That being said using a new prop like `css` is also an option:

```js
render(<button css={buttonStyle}>x</button>);
```

### A new primitive

Once style or css prop accepts an object that contains CSS, we are essentially creating a new primitive. This primitive can be passed arround, reused, overriden and React will know what to do with it. This opens a number of new doors in the future.

### Compile target

This new primitive can be targeted by compilers (babel plugins and co.). In the simplest scenario it will work without any compilation steps. With compilation it would be possible to compile for example CSS modules into something that React can take and render.

Example of CSS modules being rendered by the new primitive:

```js
import styles from "./button.css";
render(<button className={styles.button}>Hi</button>);
```

What happens right now with CSS loader:

```js
// button.js
import styles from "./button.css";
render(<button className={styles.button}>Hi</button>);
```

Compiles `button.css` to:

```js
// button.css
const createStyleNode = (id, content) => {
  const style = document.createElement("style");
  style.id = id;
  style.textContent = content;
  document.head.appendChild(style);
};

createStyleNode(
  "/src/button.css:-css",
  ".button-some-suffix {\n  color: red;\n}"
);
```

Instead, it could compile to something like this:

```js
// button.css
export default {
  button: {
    css: ".button-some-suffix {\n  color: red;\n}",
    id: "/src/button.css:-css",
  },
};
```

and

```js
// button.js
import styles from "./button.css";
render(<button style={styles.button}>Hi</button>);
```

Optionally many other compiler-based things can be enabled:

- vendor prefixing
- optimizations for compression
- linting
- other preprocessing options e.g. using PostCSS

After babel plugin the call into `css` tag function can be removed and the result of the above example can be compiled to:

```js
const buttonStyle = { css: ".css-0 { color: red; }" };
render(<button style={buttonStyle}>x</button>);
```

### Object based styles syntax

It is still possible to use object based syntax like in [JSS](https://github.com/cssinjs/jss/blob/master/docs/jss-syntax.md) or [React Native](https://reactnative.dev/docs/stylesheet). It is possible because in the end it's still going to be a CSS string which can be passed to React:

```js
const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: "center",
    alignItems: "center",
  },
}); // {container: {css: ".r-d23pfw {flex: 1; justify-content: center; align-items: center;"}}
render(
  <View style={styles.container}>
    <Text>Hi</Text>
  </View>
);
```

# Drawbacks

- we are adding a new area of responsibility to React, which will require more work to maintain it, but hopefully can be parallelized
- it inevitably will raise the discussions about how React is enforcing CSS-in-JS, so we will have to prepare a good answer
- there are several of unresolved features listed below, which will have to be addressed over time

# Alternatives

This is a very simple initial version, so it's not as feature-rich as some other CSS-in-JS solutions, but it has a potential to cover more use cases later.

It is not a replacement for react-native approach.

It is not a replacement for any entirely different approaches like SwiftUI.

# Adoption strategy

It is not a breaking change.

# How we teach this

It is a primitive interface for passing CSS to React to integrate with component's lifecycle. It can be also a way for babel plugins and compile-time processing of CSS for React components.

# Unresolved questions

- Specific algorithm for unique class names generation
- Composition
- Overrides
- Dynamic or state-based styling
- Theming
- Alternative object-based syntax
- Compatibility with react-native
