- Start Date: 2019-10-26
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Currently, setting error boundaries require usage of class components. In this RFC, I am suggesting to add error boundary functionality to functional components.

# Basic example

I suggest having an API such as this:

```jsx
const MyErrorBoundary = React.errorBoundary((props, error, stack) => {
    // This code segment is equivalent to `componentDidCatch`
    useEffect(() => {
        if (error !== null) {
            logErrorToBackend(error, stack);
        }
    }, [error]); // dependency is important here to not sent the information on every render

    // No need to derive state from error anymore. Just use the error value to do whatever you want
    if (error) {
        return <ErrorBox error={error} />
    }

    return props.children;
});
```

In class components, having a static method to derive state is in line with the overall design (all logical segments are controlled by lifecycle methods). However, in function components, the idea of "lifecycle" methods do not exist. So, in my opinion, there is no need to update the component state because the control flow is leaner in function components. Currently, I think keeping it simple as above makes the code much easier to understand. 

# Motivation

This RFC supports using functional components to create error boundaries. This way, developers can basically use functional components for all parts of the application instead of relying on class components for managing the state.

# Detailed design

Just like `forwardRef` and `memo` functions, `errorBoundary` function returns a new type:

```
function errorBoundary<Props>(render: (props: Props, error: Error, stack: ErrorStack) => React$Node) {
    // ... tests here to wrap memo and forwardRef
    return {
        $$typeof: REACT_ERROR_BOUNDARY_TYPE,
        render
    };
}
```

Additionally, a new work tag is added to React Fiber (e.g ErrorBoundary). When throwing the error, if the tag is `ErrorBoundary`, we need to update the component with the given error values (`errorInfo.value` and `errorInfo.stack`). This is all I could think of after investigating React's source code in order to match it.

# Drawbacks

Why should we *not* do this? Please consider:

- implementation cost, both in term of code size and complexity: I do not think implementing it is hard; however, I am not very familiar with React's codebase. I have 
- Can it be implemented in user space: No
- the impact on teaching people React: I think people who use functional components will be very excited to know that they can just remove class components from their codebase. As a result, I think a lot of developers will be eager to learn this feature. Additionally, it will make it easier to newcomers to learn these features because currently, when talking about Error Boundaries, they need to learn class components. Otherwise, they can just ignore class components altogether and stay and keep using function components.
- integration of this feature with other existing and planned features: Not sure
- cost of migrating existing React applications (is it a breaking change?): It is not a breaking change

# Alternatives

Currently, if this is not implemented, class components need to be used to add error boundaries.

# Adoption strategy

This is not a breaking change. Existing projects will still work because class based components are not going anywhere in the near future. However, migrating error boundary to a functional component is going to be very simple. 

# How we teach this

When designing this API, I have looked into APIs of `React.memo` and `React.forwardRef` in order to have a consistent interface when using these kinds of "helper" functions (not sure if there is a name for these kinds of functions). For example, `props` value coming first in the `render` function argument of `errorBoundary`. Additionally, the implementation for the error boundaries is not constrained by `componentDidCatch` and `getDerivedStateFromProps`. So, the developer can use existing features (e.g hooks) to perform operations based on error.

This API will need to be added to the "Error Boundaries" page.

From personal perspective, I learn about new changes by looking into Blog Post, then reading about it in the docs. So, for me this type of learning works the best. However, since existing developers are already familiar with Error Boundaries, they do not need to learn a new concept; just the new API / implementation detail.

# Unresolved questions

I am thinking that, maybe adding an additinal custom hook for `componentDidCatch` can be a useful utility to have:

```
const MyErrorBoundary = React.errorBoundary((props, error, stack) => {
    useErrorCatcher((error, stack) => {
        logErrorToBackend(error, stack);
    }, error, stack)

   // ... rest of the code
});
```

This is just a simple utility that performs the operation when error changes. Otherwise, the same logic can be written using `useEffect`.
