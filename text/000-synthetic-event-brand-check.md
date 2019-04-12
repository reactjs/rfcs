- Start Date: 2019-04-12
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

A new API `React.isSyntheticEvent(value)` which returns `true` if the value is a `SyntheticEvent` or `false` if it is anything else. 

# Basic example

```js
import React from "react"

export function useFormField<T>(defaultValue: T) {
  let [value, setValue] = React.useState(defaultValue)

  function onChange(val: T) {
    if (React.isSyntheticEvent(val)) {
      throw new Error("Expected `formField.onChange` to be called with a value for the form field, not an event")
    }
    setValue(val)
  }

  return {
    value,
    onChange
  }
}
```

# Motivation

Right now it is very difficult to say if a value is a `SyntheticEvent` or not. You have to assert the expected shape of a `SyntheticEvent` which is unlikely but still possible to break.

```js
function isSyntheticEvent(value) {
  if (typeof value !== "object" || value === null) return false
  if (typeof value.bubbles !== "boolean") return false
  if (typeof value.cancellable !== "boolean") return false
  // etc...
  return true
}
```

# Detailed design

React should export a function that looks mostly like this:

```js
export function isSyntheticEvent(value) {
  return value instanceof SyntheticEvent
}
```

# Drawbacks

- N/A ?

# Alternatives

- Exporting `SyntheticEvent` somewhere, so that others can brand check `instance SyntheticEvent` (this exposes more of the React API)

# Adoption strategy

I'll write a prollyfill which does as much shape checking as possible, that will look like this:

```js
import React from "react"

export default React.isSyntheticEvent || function isSyntheticEventPolyfill(value) {
  if (typeof value !== "object" || value === null) return false
  if (typeof value.bubbles !== "boolean") return false
  if (typeof value.cancellable !== "boolean") return false
  // etc...
  return true
}
```

This will allow React developers to depend on the library until only versions of React that support `isSyntheticEvent` are commonplace. Allowing for a smooth transition.

# How we teach this

I don't think it needs to be any more complicated than the documentation for `isValidElement()`:

https://reactjs.org/docs/react-api.html#isvalidelement

# Unresolved questions

- N/A ?
