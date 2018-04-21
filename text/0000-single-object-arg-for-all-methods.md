- Start Date: 2018-04-20
- React Issue: (leave this empty)

# Summary

Wrap the arguments of React class/instance methods with `{}`

# Basic example

```
static getDerivedStateFromProps(nextProps, prevState) -> static getDerivedStateFromProps({nextProps, prevState})
shouldComponentUpdate(nextProps, nextState) -> shouldComponentUpdate({nextProps, nextState})
componentDidUpdate(prevProps, prevState, snapshot) -> componentDidUpdate({prevProps, prevState, snapshot})
```

# Motivation

With that change, the arguments' order of the class/instance methods is no longer a barrier when contributors need to change/correct the existing class/instance method

# Drawbacks

- No matter how this will be a breaking change. But there is no need a transition becuase the existing application just simply wrap the arguments with `{}`.
- It might have performance issue but the destructuring-simple in http://incaseofstairs.com/six-speed/ does not show it.

# Adoption strategy

See Drawbacks
