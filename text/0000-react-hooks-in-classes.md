- Start Date: 2019-10-03
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Provide a new Class Based Component API that utilizes React hooks.

# Basic example

```jsx
class SomeComponent extends React.NewComponent {
    someFn = () => {
        // use this.props here
    }

    state = new State('test');
    effect1 = new Effect(this.effectFn, [this.state]);

    effectFn = () => {
        const val = this.state.getValue();

        someFn();
        // or state.setValue('some other value')
    }

    render() {
        // you can use this.state.getValue() here
        return (

        );
    }
}
```

# Motivation

In my opinion, combined with Hooks, React's composability is an implementation of [Entity-Component-System](https://en.wikipedia.org/wiki/Entity_component_system). Game Developers use this technique to set up entities (e.g a Tree or a Vehicle) in their game world. We are essentially using the same technique to set up entities (React Components) for UI. If we actually map ECS to React, this is how I see it: Entity = React Component, Hooks = Components, System = React Reconciler + Renderer.

In general, ECS systems really shine when using classes. Writing components and assigning them to entities feel natural. We are using all the class features (class fields, methods, inheritance) to abstract away what is going on behind the scenes. However, with current state of using hooks with functions, React essentially created these class features for functions. For example, a function is never recreated in a class. So, we need a `useCallback` to imitate the same behavior in function components. Same goes for `useRef` (excluding ref-ing to DOM) to imitate class member variables.

The motivation behind this RFC is that, developers typically have a certain understanding of constraints in software paradigms. I believe that using hooks in functional components add new rules to how we perceive the concept of functions. This is especially true for newcomers who start using React. I try to help newcomers with understanding concepts in React and mainly, the only confusing part for majority of use-cases is regarding hooks. It is not about "how" to do something but more about "why" certain constraints (e.g why we can't put hooks in if/else) are necessary. I have had positive experiences for explaining the "why" on these contraints by comparing them to class fields and class methods.

I think that a class based interface will provide a more intuitive interface for using hooks.

# Detailed design

Let's look at the example above. Firstly, because we are dealing with classes, the need for `useCallback` and `useRef` hooks become unnecessary -- classes already have these behaviors (class fields and methods) engrained in them.

Secondly, as you see, hooks are instances of their respective classes. Internally, all hooks are created from a base class called `Hook`. This way, React can identify if a class field is a Hook or not (e.g `clsField instanceof Hook`). Because all the logic is done in React, the hooks are essentially containers for accessing values. React just keeps track of references to these hooks to perform the necessary updates.

If we define the `State` hook, it can look something like this:

```jsx
class State extends React.Hook {
    constructor(initialValue) {
        this._value = initialValue;
    }

    register() {
        // this also has a refernce to the containing component -- coming from React.Hook
        this._setter = dispatcher.registerStateHook(this);
    }

    getValue() {
        return this._value;
    }

    setValue(val) {
        this._setter(val);
    }

    run() {
        this._value = dispatcher.getStateHookValue(this);
    }
}
```

React calls hook's `run` function to update the container.

Now, let's give a simple implementation for Effect hook:

```jsx
class Effect extends React.Hook {
    constructor(effectFn, deps) {
        this.effectFn = effectFn;
        this.deps = deps;
    }

    register() {
        dispatcher.registerEffectHook(this);
    }

    run() {
        // takes advantage of deps and effectFn
        dispatcher.callEffectHook(this);
    }
}
```

What about Effect dependencies? What if state is a dependency? How will the Effect hook work? If you check the first example, I actually passed the `State` object, instead of state value. Because we are not changing the State object, we can easily track the value of the state from the Effect hook. For example, React can do something like below when executing the effect:

```jsx
// dispatcher uses the Effect object to extract the data
// and passes it to [my guess is] Reconciler
const reconcilerCallEffectHook = (effectFn, deps) => {
    deps.forEach((dep) => {
        if (dep instanceof React.State) {
            testDependencyChange(dep.getValue());
        } else {
            testDependencyChange(dep);
        }
    })
}
```

It might be a bit tricky to set up the same idea for incoming props. To be honest, I haven't thought about the implementation of this part but using some kind of "gathering pass" to collect values from props, context, and state might actually work here.

What about custom hooks? Custom hooks are very essential for composition. I suggest having a separate base class provided to the users. We can call this class `CustomHook`. Usage would look similar to this:

```jsx
class DataFetcher extends React.CustomHook {
    _loading = new State(false);
    _data = new State([]);
    _error = new State(null);

    effect = new Effect(this.fetcherFn, []);

    constructor(url) {
        super();
        this.url = url;
    }

    fetcherFn = async () => {
        try {
            this._loading.setValue(true);
            const response = await fetch(this.url);
            const jsonData = await response.json();
            this._data.setValue(jsonData);
            this._loading.setValue(false);
        } catch (e) {
            this._error.setValue(e);
            this._loading.setValue(true);
        }
    }

    // use ES6 setter / getters for nicer access
    get loading() {
        return this._loading.getValue();
    }

    get data() {
        return this._data.getValue();
    }

    get error() {
        return this._error.getValue();
    }
}

class MyComponent extends React.NewComponent {
    fetcher = new DataFetcher(someUrl);

    render() {
        if (this.fetcher.loading) {
            return 'Loading...';
        }

        if (this.fetcher.error) {
            return 'Error';
        }

        const data = this.fetcher.data;

        return data.map(...);
    }
}
```

The definition of the custom hook is also very simple. This class sets its hooks' component member value to its own component value (typically set by `React.NewComponent`):

```jsx
class CustomHook extends React.Hook {
    register() {
        Object.keys(this).map(key => this[key] instanceof React.Hook).forEach(key => {
            const hook = this[key];
            // store this hook's component as its hooks' components
            hook.component = this.component;
            hook.register()
        })
    }

    run() {
        Object.keys(this).map(key => this[key] instanceof React.Hook).forEach(key => {
            const hook = this[key];
            hook.run()
        })
    }
}
```

# Drawbacks

I haven't checked React's internal source code to fully answer the first question; however, if there is a way to adapt the existing system for handling hooks to this logic, I think it shouldn't be as complex.

I do not know what the second point entails by "user space." Do you mean if this can be implemented outside the scope of library internals? If yes, then no. I actually tried.

In terms of teaching, I think it will be easier to explain this to developers because a lot of contraints in hooks for functional hooks are just part of classes.

Regarding integration with other existing features, I am not sure. The only problem might be with upcoming concurrent mode. However, if class based hooks are just a "wrapper" over the existing hook logic, I don't think this will affect the change.

With regards to migrating existing React application, this is not a breaking change. This is an addition. So, anyone can write hooks using classes or functions.

# Alternatives

Current alternative to this is hooks in functional components.

# Adoption strategy

Since this is an addition, I think adoption strategy would be like introduction to hooks. Class components still exist while hooks are provided as another way of writing components. This is why I called new base component as `React.NewComponent` to show that it is different than `React.Component` base class. If it is possible, we might add these features to existing `React.Component` and add an invariant that hooks cannot coexist with lifecycle methods. I think this will make it a bit confusing; however, it is still a route that is worth exploring.

# How we teach this

Terminologies are already in place for hooks. Hooks documentation will need to be updated to accommodate for hooks in both functional components and class components. 

# Unresolved questions

I have provided the general pattern on how hooks will operate. However, each individual hook's design should be designed. I have written rough sketches for State and Effect hooks; plus Custom Hooks. The following additional hooks need to be designed: Reducer, Ref (I would call it NodeRef to make it more specific), Memo, ImperativeHandle, LayoutEffect, and DebugValue. 