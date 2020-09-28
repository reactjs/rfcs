## Summary
I would like to suggest a unify API for creating a global, contextual and local states. 
This addition to React will hopefully completely mitigate the need for 3rd party state management libraries, and will solve some of the biggest problems
of hooked functions- depending on call order for inferring the state, and inability to share state with class components.

## Motivation
Currently, there are 2 ways to define local state (this.state for classes and useState for function components), there are two different API's for creating 
and consuming contextual state (React.Context), depending if you are using hooks of class components, and there is no way for creating a global state.

Currently if you are using React Native with a native navigation solution, the only way to share state between screen is by using a 3rd party library solution for creating a global state (aka store), because different screens don't 
share the same React root. Instead of suggesting a a new API for creating a global state with React, I think we can unify the way we are creating a state, 
thus addressing the problem of creating global state and solving some of the problems that hooks introduced to the framework, all of this by keeping the API 
small and concise.

## API Summary 
React will add 3 types of state object, all with the exact same API:

```javascript
const globalState = React.GlobalState(initialState, actions);
const contextualTest = React.Context(initialState, actions});
const localState = React.State(initialState, actions});
```

As you can see, the api is the same, and the separation is for readabiliy purposes only. We could introduce A single `React.state()` api, and the type of state will be
defined by how you use it, But I found it more safe to introduce a new command for every type of state.

### Example:
```javascript
const someState = React.GlobalState({count: 0}, (state) => ({
       getCount: () => state.count,
       increaseCount: () => state.count += 1
}));
console.log(someState.getCount()); // prints 0;
someState.increaseCount();
console.log(someState.getCount()); // prints 1;
```

You can see that the type of `someState` is actually what we return from the function in the second parameter (the actions generator function). This way, the only
way to interact with the state is by using the actions.

Here is the formal declaration of such a state:
```typescript
type ActionsObject<T> = {[key: string] : (state: T) => any};
function xState<T>(initialState: T, actionGenerator: (state: T) => ActionsObject<T>): ReturnType<actionGenerator>;
```

## Usage Example
let's see how we can use those 3 kind of states:

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
