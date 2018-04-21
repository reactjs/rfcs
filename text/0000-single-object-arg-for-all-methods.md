- Start Date: 2018-04-20
- React Issue: (leave this empty)

# Summary

Wrap the arguments of React class/instance methods with curly brace

# Basic example

`static getDerivedStateFromProps(nextProps, prevState) -> static getDerivedStateFromProps({nextProps, prevState})`
`shouldComponentUpdate(nextProps, nextState) -> shouldComponentUpdate({nextProps, nextState})`
`componentDidUpdate(prevProps, prevState, snapshot) -> componentDidUpdate({prevProps, prevState, snapshot})`

# Motivation

With that change, The arguments' order of the class/instance method is no longer a barrier when contributors need to change/correct the existing class/method

# Drawbacks

No matter how this will be a breaking change. But there is no need a transition becuase the existing application just simply wrap the arguments with {}.
The destructuring-simple in http://incaseofstairs.com/six-speed/ does not show any performance issue.

# Adoption strategy

See Drawbacks
