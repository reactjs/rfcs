- Start Date: 2018-04-27
- RFC PR:
- React Issue:

# Summary

This RFC seeks to add the capability for a component subtree to opt out of rehydration diffing.

# Basic example

I propose adding a new lifecycle method: `shouldComponentHydrate`. Returning `false` from this method should cause React to opt out of reconciliation during rehydration and leave the subtree as-is.

```jsx
function getComponent() {
    return (
        typeof window === 'undefined'
            ? require('./foo').default
            : import('./foo').then(mdl => mdl.default)
    );
}

class CodeSplittingComponent extends React.Component {
    constructor() {
        this.state = {
            component: getComponent(),
        };

        if (this.state.component instanceof Promise) {
            this.state.component.then(ReactClass => this.setState({ component: ReactClass }));
        }
    }

    shouldComponentHydrate() {
        return false;
    }

    /**
     * Block re-rendering until the component is resolved and ready.
     */
    shouldComponentUpdate(props, state) {
        return !(state.component instanceof Promise);
    }

    render() {
        return (
            <this.state.component />
        );
    }
}
```

# Motivation

The basic goal here is to allow something to be rendered by the server and remain untouched by React until the component enqueues a subsequent rerender via prop or state change.

When working in a code splitting context, this becomes very important because what is immediately available on the server may not be on the client. A specific example of a typical SSR + code splitting flow:

1. ReactDOMServer renders a complete page to string and sends it to the browser via some means.

2. React in the browser rehydrates, but one or more component children in the tree are meant to be asynchronously-loaded in the browser context.

3. When the code splitting component is mounted, this temporarily causes the pre-rendered subtree to be discarded since the asynchronously-loaded child has not yet been downloaded.

4. When the child JS downloads, the code splitting component then enqueues a re-render and the child appears again.

Note the UX problem in step 3: pre-rendered HTML ends up being thrown out because there is no current way to opt out of reconciliation during rehydration.

# Detailed design

My proposal is to add a new lifecycle method called `shouldComponentHydrate(props)`. It should act similarly to `shouldComponentUpdate`, but be run during the rehydration phase. If `false` is returned from the method, React should halt reconciliation on this part of the subtree and move on.

When returning `false` from `shouldComponentUpdate`, `componentDidMount` should not be fired until a subsequent render causes the component subtree to be initialized for the first time.

Note that `shouldComponentHydrate` does not prevent future re-renders. It should only opt out of rehydration. Subsequent renders due to a change in props or state would need to be managed via `shouldComponentUpdate` if further reconciliation control is desired.

# Drawbacks

Some added complexity to how reconciliation works in the rehydration context.

# Alternatives

TBD

# Adoption strategy

This is an entirely new feature, so backward compatibility is not a concern. It's purely an opt-in optimization for some specific use cases that are not currently solvable in userland.

# How we teach this

Adding it to the React documentation alongside the other lifecycle methods should be sufficient. A blog post introducing it would probably also make sense.
