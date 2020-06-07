- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

The terminology used in the document:
- Logical block - a separated object or function  (for example: mixin, HOC, React hook) for component code reuse.
- Behaviour - my variant of logical block.
- Programming entity - for example: class instance, React component, logical block.
- ECS - Entity Component System pattern in game programming.
- Entity Component-Behaviour (non official name, this name used only in the document) - variation of ECS where a custom logic is written in a logical block (Component-Behaviour) of Entity, but not in the System. 
- Entity (in ECS and Entity Component-Behaviour approaches) - container for logical blocks.      

**Here are some related suggestions:**   
**1. Make component a configurable container without hardcoded render function and hardcoded custom logic.**   
**2. Add ability adding and removing logical blocks in components (at least before component creating).**

I think that is it mistake for complex components  and 3rd-party components to write logic and jsx code in a component body. More flexible way - a component is only a container for objects with logic. In the **Detailed design** section I provided my examples for react and links from game programming area, where like approaches is actively used about 20 years. I did examples using class-based components. Partially it can be applied to functional components.

# Basic example

Below are several possible implementations:   
1.1. 
```jsx
class CounterBehaviour extend BaseBehaviour {
  defaultState: { count: 0 }
  passedToRender: { 
    setCount: (this.setState({count: value})); 
  }
}
 
const CounterComponent = { 
  behaviours: [CounterBehaviour],
  render: ( {count, setCount}) => ( // render can be replaced in runtime and declared an outside a component
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>
        Click me
      </button>
    </div>
  )
}
```

1.2. Or (due to complexity of implementation) instead ```const CounterComponent = configObject;```
can be implemented variant with creating class-based component:
```jsx
const CounterComponent = createContainer(‘CounterComponent’, configObject);  
```

2 Variant with hooks (theoretical). Only for my second suggestion and partially for first suggestion.
A few limitations and possibilities will be added to react components and hooks:
   - component can use only custom hooks and jsx code; 
   - only custom hooks can use any hooks;
   - custom hooks can add or remove another custom hooks; 

# Motivation
The main motivations:
- Current 3rd-party components has not very good extensibility and compatibility of their functionality with each other.
- More flexible code and better composition. You will can to change logic of ready component without changing view code.
 
**I. Motivations for “Make component a configurable container without hardcoded render function and custom logic”.**    
1. Two sources of custom logic (component, logical block) instead of one to allow write dirty code. There is no splitting code by layers. The programmer is forced to choose where to write a logic. Even experienced developers first choose a simpler way, but not a flexible - “writing logic in component”. Next, in the case of a complex component, they will spend time moving logic outside the complex component. This way is good for small components. But this way is not suitable for complex components and component libraries.

   In React applications, it is customary to split a project structure into multiple layers (actions, store, component). At the same time, it is strange that in React is not customary to split a component logic and jsx code into 2 different layers. As a result, React has 2 different programming entities (component, custom hook) in which most responsibilities coincide.

2. In my mind the Single Responsibility Principle of SOLID need strives to reduce the number of responsibilities. The react components has too much responsibilities:
   - container for logical blocks (hooks);
   - custom logic inside component;
   - life cycle events handling;
   - component state management;
   - view (JSX code);   
   
   For simple components this is an unnecessary complication. Blind following of SOLID to complicate an application.
   But it is useful for large systems where you need to build complex components by combining logical blocks.
   
3. React components do not satisfy the second SOLID principle - “Open–Closed principle”. If you need to extend component then in some cases you must to change a code inside of component. For 3rd-party components this is impossible.
4. If move the render function outside of component then one render function can be used in several components even if the props are different (you can transform props before passing to render).
5. More clean code in render function. All used variables are visible in render function parameters:   
```render: ({ count, setCount }) => {...}```

**II. Motivation for “Add ability adding and removing logical blocks in components”**  
1. This will allow you to modify 3rd party components instead of writing a suitable component from scratch. Current react components (including components with hooks) are not very flexible. If you need to slightly modify the 3rd party component, but it does not support this features then you have to create own component that is often difficult and take many time. I have come across this situation several times. At current time it is impossible remove incorrect/unnecessary hook from component or adding new hook.
2. In case of compatibility problems, it lets to add an ability to remove incompatible functionality from the component and replace it with your own.
3. Adding and removing logical blocks allows to modify the any behaviour of a component without changing a code inside of component. It will allow components do satisfy the second SOLID principle - “Open–Closed principle”.

# Detailed design (part 1)
**Additional important and useful information.**    
I took these ideas from from game programming. I used it for several years when creating games and games UI using the Unity3D engine.   
To familiarize yourself: there are 2 good patterns in game programming for code reuse - Entity Component System (ECS) and variation of Entity Component System, where component with behaviour used instead System. They have been successfully used for about 20 years.   
**Note:** Component in gamedev can be any object (script, collider, 3d mesh) that can be added to the Entity. For simplicity I consider only variant where component is a script.   
**1. Entity Component System (ECS).** It Is data driven approach with better performance, but more complicated.   
   - **Entity** - container for components. You cannot write your own code in the container, only add or remove  components.   
   - **Component** - object with data. Without logic.   
   - **System** - is typically an implementation that iteratively operates on a group of entities that share a specific set of components.   
   
**useful links:**   
- https://github.com/junkdog/artemis-odb/wiki/Introduction-to-Entity-Systems
- https://www.raywenderlich.com/2806-introduction-to-component-based-architecture-in-games#toc-anchor-009    
     
**2. Entity Component-Behaviour** (non official name, this name used only in the document)
The main idea - a programming entity (game unit, button) is composed of 2 classes (Entity and Component).    
- **Entity** - a container for storing a components list. You cannot write your own code in an entity, only add or remove components.   
- **Component** - logical block (like custom react hook) for custom logic.    

**useful links:**   
- https://gameprogrammingpatterns.com/component.html (find text “class ContainerObject” in the page for example)
- https://docs.unity3d.com/Manual/Components.html
- https://www.raywenderlich.com/2806-introduction-to-component-based-architecture-in-games#toc-anchor-007

In document by link below  I described only the second approach, because I did not work with ECS.
In addition the second approach can be implemented in user space.

[Part 2 (implementation description)](0000-entity-component-behaviour-approach-implementation-description.md)

# Drawbacks

- **implementation cost, both in term of code size and complexity**   
My implementation has code size about 300 rows for unoptimized implementation in user space for class-based components.   
I can not estimate the complexity of implementation in the react core. I think this is equivalent to implementation of react.Component class with lifecycle and state.
- **whether the proposed feature can be implemented in user space**   
It's can be implemented in user space (with worse performance). 
- **the impact on teaching people React**   
At the moment it is difficult for me to answer to it.
- **integration of this feature with other existing and planned features**   
Depends on the chosen implementation. At the initial stage, it is better to implement with using the existing API or with small changes in existing API. Preferably in a separated library (if to implement with using the existing API like my solution).   
- **cost of migrating existing React applications (is it a breaking change?)**   
Like react hooks. It is a not breaking change.

##### Other Drawbacks 
- Possible a decreasing performance when using the Entity Component-Behaviour approach.
- Possible difficulties with implementation in functional components.
- (in my implementation) Small components looks more complexity by comparing with small functional components.
- (in my implementation) Another variation of logical blocks and code reuse. Already exist mixins, hoc, custom hooks.
- (in my implementation) Another structure of component.

# Alternatives
For  “Add ability adding and removing logical blocks in components (at least before component creating)”:
1. To add ```addHook```, ```removeHook``` functions.
2. To add a few limitations and possibilities to react components and hooks:
   - component can use only custom hooks.  
   Or to add ```useBehaviours``` hook:   
    ```useBehaviours([Behaviour1, Behaviour2], props)```  
   - only custom hooks can use any hooks; 
   - custom hooks can add or remove another custom hooks. 

# Adoption strategy
Depends on the chosen implementation.
If you choose the full implementation I think adoption strategy would be like introduction to hooks.

# How we teach this
- **Would the acceptance of this proposal mean the React documentation must be re-organized or altered?**  
Yes, the documentation should be expanded.   
- **Does it change how React is taught to new developers at any level?**   
Yes, like custom react hooks   
- **How should this feature be taught to existing React developers?**   
Like custom react hooks    

# Unresolved questions
I have provided only general conception with class-based examples. 

