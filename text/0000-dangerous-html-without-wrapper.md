- Start Date: 2019-11-06
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

ReactDOM.RawHTML is a new component that lets you dangerously set innerHtml without a wrapper element.

# Basic example

```jsx
import * as ReactDOM from 'react-dom';

function Layout(props){
  return (
    <html>
    <head>
      <ReactDOM.RawHTML dangerouslySetInnerHTML={{ __html: props.headerHTML }} />
    </head>
    <body>
      hello
    </body>
    </html>
  )
}

ReactDOM.renderToString(
  <Layout headerHTML={`<script src="/app.js"></script> <link href="/styles.css" />`} />
)
```

```html
<html><head><script src="/app.js"></script> <link href="/styles.css" /></head><body>hello</body></html>
```

Uses the same `dangerouslySetInnerHTML` keyword as DOM elements, but when rendered does not include any wrapping DOM element.

# Motivation

Today in React, one must wrap dangerous html in an element ```<div dangerouslySetInnerHtml={{__html: `<span>ok</span>`}} />```. 
This results in an unwanted wrapper element, which in the head tag is illegal.


# Detailed design

`RawHTML` is a new component that lives in ReactDOM.

`RawHTML` only accepts a prop of `dangerouslySetInnerHTML`.

The prop type of `dangerouslySetInnerHTML` is an object with key `__html` and value `string`.

When rendered the output is the content of `__html`.


# Drawbacks

- An additional way to set dangerous html.
- Discovering the API might be difficult, folks might look to dangerouslySetInnerHTML on a Fragment first.


# Alternatives

- https://github.com/remarkablemark/html-react-parser


# Adoption strategy

- Non-breaking
- Could coordinate with html-react-parser to have them help distribute the update


# How we teach this

- There have been issues filed for renaming dangerouslySetInnerHTML, but I think consistency would be prefered.
- Adding to the docs near https://reactjs.org/docs/dom-elements.html#dangerouslysetinnerhtml


# Unresolved questions

- Which package does this component belong in?
- Are there repurcussions for comparisons across renders? 

