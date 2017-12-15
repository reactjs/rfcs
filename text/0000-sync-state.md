- Start Date: (2017-12-15)
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Add support for synchronous state in React:
(1) Either replace the existing asynchronous ```setState``` method, or
(2) Add a new function which should synchronously set state.

# Basic example
```this.setState({val: newVal}) // Now react should synchronously update state instead of asynchronously```

or

```this.setStateSync({val: newVal}) // Now react should synchronously update state instead of asynchronously```


# Motivation

The motivation is that it is much easier to reason about application state
when the function which modifies state does this synchronously.

Nobody will disagree that react's asynchronous ```setState``` has caused lot 
of confusion among developers.

Even given the solution react has - like ```setState``` with a function(which gives
access to a so called "pending state"), still
doesn't prove that there can't be any gotchas (or edge cases) when managing state
asynchronously.

Ask people why are they using mobx for example.

# Drawbacks

Honestly I can't think of any drawbacks which would outweigh the advantages that
synchronous ```setState``` would have.


# Adoption strategy

If you directly change the existing ```setState``` with a synchronous one this seems
like a breaking change, if you add a new synchronous ```setState``` function this doesn't seem to be a 
breaking change.

# How we teach this

It will be much easier to teach about it than the existing asynchronous ```setState```.

 
