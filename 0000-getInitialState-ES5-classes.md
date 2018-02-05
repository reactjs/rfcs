- Start Date: 2018-02-05
- RFC PR: 
- React Issue: 

# Summary

Make `getInitialState` method to work for ES6 classes not just for the classes created with `React.createClass`.


# Basic example
If `getInitialState` is defined it should be called for ES6 class after the class constructor.

```
class ExampleBaseComponent extends React.Component {
  getInitialState() {
    // Called after a component is instantiated.
    // Return an initial state.
    return state;    
  }

class ExampleComponent extends ExampleBaseComponent {
    getInitialState() {
        const state = super.getInitialState();
        // build part of the state related to the ExampleComponent here
    }
}

```

# Motivation

We need а reliable way to build the initial state for ES6 classes.
Currently, this is limited to setting initial `this.state` in the constructor. We need to have a function called outside of the constructor chain to build the initial state for the class hierarchy instead.

We are using this approach in [resub](https://github.com/Microsoft/ReSub) framework to provide automated subscription system that makes developers not need to understand multiple state changes, manual subscriptions etc.
Think of it as react-style `render()` function for building your state. As such, we need to build an initial `this.state`.` 
However, if you do it in the constructor for the component (this.state = this._buildState()), then all private member variables of your component aren't initialized (since they won't be initialized until after the `super()`

To overcome this limitation, we have `this.setState(this._buildState())` in `componentWillMount` lifecycle method now.

As Dan Abramov mentioned [here](https://github.com/facebook/react/issues/11847#issuecomment-351534841) this approach might cause some problems in future as there are plans to change how `componentWillMount` going to work.
[getDerivedStateFromProps](https://github.com/reactjs/rfcs/blob/master/text/0006-static-lifecycle-methods.md#detailed-design) is very similar to what we need, but as it is static method it doesn't solve the problem.

If you'd make `getInitialState` method work for vanilla ES classes the same way it works with classes created by `React.createClass` it is going to be win-win.

# Detailed design
- In case there is an implementation of the `getInitialState` for the ES class this method should be called to build `this.state` after class constructor call, so the instance of the class is properly created.
- We should stop warning about this method presence in ES classes.
- We should warn if ES class has both `this.state` passed in the constructor and the `getInitialState` implemented as the second will override the former.

# Drawbacks
  - There is a new `getDerivedStateFromProps` which is static. It might be easy to confuse it with new `getInitialState`.

# Alternatives
  - Make `getDerivedStateFromProps` not static and pass an argument `initialBuild: bool` there.

# Adoption strategy

It shouldn't be a breaking change.

# How we teach this

I don't think we should teach it as in general inheritance is [not recommended](https://reactjs.org/docs/composition-vs-inheritance.html) in React.

# Unresolved questions

