- Start Date: 2020-09-28
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

I would like to suggest a unify API for creating a global, contextual and local states. 
This addition to React will hopefully completely mitigate the need for 3rd party state management libraries, and will solve some of the biggest problems
of hooked functions- hooks depends on call order for inferring the state, and they cannot be shared with class components.

# Basic example

```javascript
/**
const globalState = React.GlobalState(initialState, actions);
const contextualTest = React.Context(initialState, actions});
const localState = React.State(initialState, actions});
*/
//edxample: 
const someState = React.GlobalState({count: 0}, (state) => ({
       getCount: () => state.count,
       increaseCount: () => state.count += 1
}));
console.log(someState.getCount()); // prints 0;
someState.increaseCount();
console.log(someState.getCount()); // prints 1;
```

As you can see, `React.GlobalState` returns the actions object, this way, the only way to interact with the state is by using the actions.

# Motivation

Currently, there are 2 ways to define local state (this.state for classes and useState for function components), there are two different API's for creating and consuming contextual state (React.Context), depending on if you are using hooks or class components, and **there is no way for creating a global state**.

If you are using React Native with a native navigation solution, **the only way to share state between screens is by using a 3rd party library** solution for creating a global state (aka store), because different screens don't 
share the same React root. Instead of suggesting a new API for creating a global state with React, I think we can unify the way we are creating a state, thus addressing the problem of creating global state and solving some of the problems that hooks introduced to the framework, all of this by keeping the API 
small and concise.

# Detailed design

React will add 3 types of state object, all with the exact same API:

```javascript
const globalState = React.GlobalState(initialState, actions);
const contextualTest = React.Context(initialState, actions});
const localState = React.State(initialState, actions});
```

As you can see, the api is the same, and the separation is for readability purposes only. We could introduce a single `React.state()` api, and the type of the state (global, contextual or local) will be defined by how you use it, But I find it safer to introduce a separate command for every type of state.


Here is the formal declaration of such a state:
```typescript
type ActionsObject<T> = {[key: string] : (state: T) => any};
function State<T>(initialState: T, actionGenerator: (state: T) => ActionsObject<T>): ReturnType<actionGenerator>;
```


Let's see how we will use those 3 types of state:

### Global State

inside store.js
```javascript
import React from 'react';
export const store = React.GlobalState({name: 'bob'}, (state) => ({
    getName: () => state.name,
    setName: (name) => state.name = name
});
```
usage: 
```javascript
import {store} from './store';
// class component example:
class Foo extends React.Component {
   render(){
     return (
        <View>
           <p>{store.getName()}</p>
           <div onClick={() => store.setName('alice')}></div>
        </View> 
      );
    }
}
// function component example:
function Bar() {
     return (
        <View>
           <p>{store.getName()}</p>
           <div onClick={() => store.setName('danny')}></div>
        </View> 
      );
}
```

In this example, both Foo and Bar will always present the same name, because they are using the same global state. when Bar will change the name to danny, 
Foo will immediately re-render and will present danny also.

### Contextual State

inside some contextual store:
```javascript
//inside someContext.js
import React from 'react';
export const someContext = React.Context({name: 'bob'}, (state) => ({
    getName: () => state.name,
    setName: (name) => state.name = name
});
```
usage: 
```javascript
import {someContext} from './someContext';
function ProfileCard() {
     return (
        <View>
           <p>{someContext.getName()}</p>
           <div onClick={() => someContext.setName('alice')}></div>
        </View> 
      );
}
```

As you can see, the usage of contextual state and global state is exactly the same. The only difference is that when we use contextual state, React will make sure that this context was provided by some component upper in the hierarchy tree, and retrieve for us the correct instance of the state:
```javascript
import {ProvideContext} from 'react';
import {someContext} from './someContext';
function ProfileList() {
   return (
     <ProvideContext context={someContext}>
        <ProfileCard/>
     </ProvideContext>    
   )
}
```

So there is a little bit of magic here. We are using the context as a global object, but behind the scenes, React will make sure that we are interacting with the correct instance. It's the exact same piece of magic that we have with UseState for hooks.

### Local State

```javascript
import React from 'react';
export const localState = React.LocalState({name: 'bob'}, (state) => ({
    getName: () => state.name,
    setName: (name) => state.name = name
});

//class example
class Foo extends React.Component {
     return (
        <View>
           <p>{localState.getName()}</p>
           <div onClick={() => localState.setName('alice')}></div>
        </View> 
      );
}

//function example:
function Bar() {
     return (
        <View>
           <p>{localState.getName()}</p>
           <div onClick={() => localState.setName('alice')}></div>
        </View> 
      );
}
```

Again, same API, same magic behind the scenes that makes each component get the correct local state. What's interesting here, is that we can use as many local State objects as we want, and we don't need to pay attention to the call order, because we can uniquely identify the states, so no need for weird rules like "don't put custom hooks inside an `if` statement".


# Additional Api
### Support effects in local state:
In order to fully support the capabilities of hooks, we need to somehow mimic `useEffect`. Here is one suggestion:
```javascript
const localState = React.LocalState(initialState, actions, effects);
```
The effects props is a function that receives the state, and also the props of the host component (the component that is currently using the local state). This function will be called on every render (and on unMount). The return value will be used as a cleanup, just like `useEffect`:
```javascript
(state, props) => {
   subscribe();
   return () => unsubscribe(); 
};
```
This will allow the creation of an hooked state, thus achieving the same behavior of `useState` + `useEffect` encapsulated together. Personally, I am not a big fan of such a pattern, but in order to achieve feature parity with the current hooks implementation, we have to allow something like this for local states.
We also need to think about a convenient API for replacing the dependency array.


# Implementation and performance
We should mimic the exact same performance optimizations of Mobx- For every component render, React will take a note for all the state properties that have been accessed, and will re-render the component on every change of relevant props.


# Drawbacks

* The API is a bit magical, but it's the same magic of Hooks (keeping track of the currently rendered component);
* The community is just starting to get used to the hooks API, and this API will again change some of the most basic concepts of React - the state. I don't think that this drawback should stop us from moving forward though, unifying the way we are using state should be one of React's top priorities.
This is not the ideal way of moving forward, but I really think that this step is needed in order to clean some of the problems of hooks, and the lack of global state solutions.

# Adoption strategy

This proposal should be fully backward compatible and it will be completely up to the users if they want to start using the unified syntax.


# Unresolved questions

* Should we add the suggested support for effects in local state? If so, we need to think of a good way for replacing the dependency array.
* The `provideContext` API is not complete yet. we need to think of a way to provide multiple context at the same time, while allowing overriding the initial state.

