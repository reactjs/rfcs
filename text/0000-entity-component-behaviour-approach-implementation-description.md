# Detailed design (part 2)
I do not propose a concrete implementation. You can to implement it using objects, classes API, hooks API.
Here is a description of my implementation with using class-based React API.
 
Link to my implementation with examples using the Entity Component-Behaviour approach:   
https://codesandbox.io/s/entity-component-behaviour-approach-in-react-zo2yy
 
**All code of implementation is inside the `core` folder.**   
**Other folders contain only examples.**
 
**Main programming entities:**
- `Container` component;
- `config` (configurable object). Used in `Container` components; 
- `Behaviour` logical block;
- `render` function.

**Data flow diagram:**

![](https://raw.githubusercontent.com/sergeysibara/entity-component-behaviour-approach-in-react/master/content/data-flow.png)
___
 
This document also provides examples of solutions for some react problems and some useful features, such as :
- creating `Container` for logical blocks (`Description` paragraph 1-3)
- moving `render` function outside component (`Description` paragraph 6)
- creating logical blocks (`Description` paragraph 9-11)
- removing/adding logical blocks in runtime (`Description` paragraph 8)
- props grouping by logical blocks (`Description` paragraph 12)
- code reuse like Vue directives - adding logical block through prop in JSX (`Description` paragraph 12-13)

## Description
**1.** I created the `ContainerComponent` class which inherits from `React.Component` that to change the behavior of all future custom components.   
```jsx
ContainerComponent extends React.Component
```

**2.** To avoid writing excess code (excess inheritance code and constructor code) I made a function to create custom components - `createContainerComponent`.
```jsx
const MyComponent = createContainerComponent(‘MyComponent’, config);
```

**3.** All necessary component parts are set in `config` object.

**4.** For to write custom logic like custom react hooks used `Behaviour` objects.
Each container has own Behaviors list.

**5.** Container send lifecycle function calls and their arguments to all own behaviors.   
Example from `createContainerComponent.js`:
```jsx
 componentDidMount() {
   this.callMethodInAllBehaviours(“componentDidMount”);
 } 
 
 callMethodInAllBehaviours(funcName, args = []) {
   this.behaviourList.forEach(beh => {
     if (beh[funcName]) {
       beh[funcName](...args);
     }
   });
 };
```

**6.** I overrided render method in `ContainerComponent` base class.   
Shortened example from `createContainerComponent.js`:
```jsx
 render() {
   const mapToRenderData = this.config.mapToRenderData;
   return this.config.render({
     props: this.props,
     ...mapToRenderData(this)
   });
 };
``` 
This allowed move render outside a component and added ability to replace `render` at runtime.   
`mapToRenderData` gives a programmer control over the data before passing data to `render` function.
 
**7.** The above steps allowed to change the component structure making it more flexible.
 Example:
 ```jsx
const counterRender = ({ count, setCount }) => (
    <>
      <h3>Counter Example</h3>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>Click me</button>
    </>
  ),
  
 createContainerComponent("CounterComponent", {
  behaviours: [ // optional
    // ...
  ],
  render: counterRender, // optional
  mapToRenderData: mapToRenderDataFunction, // optional
 });
```

 Lifecycle methods, constructor, render, state removed from the component. Custom logic are located not in the component. Custom logic located only in behavior and a bit in mapToRenderData function.
 There are only a few options in the config.
 There are several more options in the elements of the behaviors array, but more on that later.
  
 All data used in the render function is passed through the function parameters.

**8.** I added next methods to the `ContainerComponent` base class:   
`addBehaviour(behaviour, props, initData)`  
`removeBehaviour(behaviourInstance)`

They allow to add and to remove behaviors in runtime.   
Now a programmer can to change almost any part of a ready component including 3rd-party components.

**9.** Behaviours   
Using behaviours like using mixins.   
Unlike mixins, all data and methods of behaviors are isolated from the component and from each other.   
But they can refer to each other through a component reference and a references to all component behaviours.

Struct of my behaviours:
- init method; // instead constructor
- life cycle methods;
- name field; // name of behaviour; (editable)
- type field; // type of behaviour; (readonly)
- state; // getter for getting state object;
- setState; // like setState in react
- useState; // syntactic sugar for imitation useState from react hooks
- defaultState object;
- passedToRender object;
- ownProps getter; // to get grouped props only for current behavior
- mapToRenderData; // to pass data to render function of component 

I created the `BaseBehaviour` class. Its functionality is like react class-based react component.   
It has the same lifecycle methods and it has his own state. But it has several differences:
- another API;
- additional lifecycle methods;
- `init` method instead constructor;
- has no render method;
- has `passedToRender` object that can contains any data and functions for passing to render function;
- instead render method used `mapToRenderData` method  for passing data to render function. By default passed content of `state` and `passedToRender` objects.

**Motivation for passedToRender:**   
It avoids the functions re-creation on every component update and also avoids excess code for caching functions and data.
For example:   
```jsx
passedToRender: {
  value1: 10,
  onChange: ()=> { ... }, // a link to this function will be passed always. New functions will not be created on component updating.
}
```

**10.** `init` method    
The motivation for using init instead of the constructor:  
In parent constructor `this` stores data and methods defined only in the parent class. We cannot get overridden child fields in parent constructor. Because of this, in each child need to override the constructor for proper operation, that is inconvenient. Init calls after component creating, so `this` in `init` contains overridden fields and methods.

**11.** Every behaviour has own state    
I just imitate separated states. In fact the component stores the states (common state) of all own behaviors.
```jsx
setState(stateObject) {
   this.component.setState(() => {
     return {
       ...this.component.state,
       [this.name]: stateObject, // this.name - name of behaviour
     };
   });
 }
```
I think it is right that the state is stored in the component. This feature allows a behaviours group to has a common state.


**12.** `ownProps` to get only props of current behaviour   
In `BaseBehaviour` class I made a small solution for grouping props by behaviours.
```jsx
get ownProps() {
   const propBehaviourName = `bh-${this.name}`;
   return this.component.props[propBehaviourName];
 }
```
using example:   
```jsx
 <Form bh-bindModel={customModel} bh-formController={ {onSubmit: customAction }} /> 
```
**Motivation for inbox solution of props grouping:**   
This is an small, easily implemented and useful feature for fix “props hell” (when many props passed to component without grouping, it’s hard to understand to which hook, HOC, child component they are applied).    

Most developers do not try to solve this problem, which makes their code less readable.   
Link to “props hell” example in popular real project:   
https://github.com/marmelab/react-admin/blob/master/packages/ra-ui-materialui/src/form/SimpleForm.js

This feature also can be used in exist hooks api:
```jsx
const useOwnProps = (componentProps, hookName) => {
  return useMemo(() => {
    const propHookName = `h-${hookName}`;
    return componentProps[propHookName];
  }, [componentProps]);
};

const useMyCustomHook = componentProps => {
  const ownProps = useOwnProps(componentProps, "MyCustomHook");
  // ...
};

const MyComponent = props => {
  useMyCustomHook(props);
  // ...
};

// using: 
<MyComponent h-MyCustomHook={{ a: 1, b: 2 }} />
```

**13.** Adding default behaviours through props.   
Example:
```jsx
<CounterComponent defaultBehaviours={[{behaviour: CounterBehaviour, {count: 0} ]}>
```
it is equivalent:
```jsx
const createContainerComponent(‘MyComponent’,  { 
  behaviours: [{behaviour: CounterBehaviour, {count: 0}}],
  ...
 }
);
```

This is like custom directives in Vue and a bit like using custom hooks as below:
```jsx 
<input {...useInput("fieldName")} />
```

Directives are a very useful feature that allows you optionally to add logical blocks to a component. Combining directives, you can to use one component with different behaviors instead of creating new kinds of components. This is more efficient than creating multiple components.   
Directive - this is just a way to use a logical block through props. No need to invent a new programming entity, as done in Vue. In Vue this is an over complication.


**14.** Each behavior contains the `mapToRenderData` method for passing fields and methods of `passedToRender` and `state` objects to the `render` function in component.   
Example from `BaseBehaviour.js`:
```jsx 
 mapToRenderData() {
   return {
     ...this.passedToRender,
     ...this.state
   };
 }
```

**15.** `wrapRenderData`   
The `wrapRenderData` function can be used for data transform from concrete behaviour. 
Called after `mapToRenderData` of behaviour inside `mapToRenderData` of component
```jsx 
const FormContent = createContainerComponent("FormContent", {
 behaviours: [
   {
     behaviour: FormBeh,
     wrapRenderData: (data) => {
        return { firstName: data.first_name; lastName: data.last_name }; // example of data formatting for using in render function
   },
 ],
 render: ({firstName, lastName}) = … 
});
```
`wrapRenderData` used if you need to perform some additional data formatting before passing it to the component. Usually, just overriding the `mapToRenderData` method in the behavior is enough. But if you want to make a common function for different behaviours and different components, you can to use `wrapRenderData`.

**16.** Parameters of the `init` method of a `behaviour`. Passing initialization data into `behaviour`. Description of the behaviors options in the `config` object of the component.

Description 'initData' fields:  
```jsx 
createContainerComponent("ConcreteComponent", {
 behaviours: [
   {
     behaviour: Behaviour1, // (required)
     initData: { // any custom data that will passed to init method of a behaviour.
       name: “MyBehaviourName”, // name for this behaviour. (optional)
       defaultState: {}, // data that will passed in defaultState of Behaviour. (optional)
       wrapRenderData: wrapperFunction; // (optional)
       //any custom data
     }
   },
   { behaviour: Behaviour2 }
 ],
 ```
`initData` in `init` method of behaviour:
```jsx 
init(component, props, initData) { … };
```

**17.** `mapToRenderData` option in `config` object of component.   
The `mapToRenderData` is like to 'optionMergeStrategies' в Vue:   
https://vuejs.org/v2/guide/mixins.html#Custom-Option-Merge-Strategies
 
Usually `mapToRenderData` no need to use. In this case all data from a behaviours will merged like mixins.   
To avoid a names conflict you can to use the prepared function `mapToIsolatedRenderDataObjects` from `mapToRenderDataStrategies.js` or to write own function.   
```jsx 
import { mapToIsolatedRenderDataObjects } from "../core/mapToRenderDataStrategies";

createContainerComponent("ConcreteComponent", {
 render: ...
 behaviours: ... // multiple behaviours   
 mapToRenderData: mapToIsolatedRenderDataObjects,
}
```

then in the `render` instead of simple arguments like
```jsx 
const render = ({
 x,
 y,
 isLoading,
 title,
 body,
 props
}) => (
```
will passed objects (the object name is taken from the `name` field of a Behaviour), for example:
```jsx 
const render = ({
 behaviour1Data,
 behaviour2Data,
 props
}) => (
```

For more complex cases, you can to write own mapToRenderData function.
