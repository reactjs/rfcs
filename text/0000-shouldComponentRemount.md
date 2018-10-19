- Start Date: 2018-10-17
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

_**Draft**_

# Summary

This is a proposal to add a new lifecycle method, `shouldComponentRemount`.
Returning `true` from this method would be equivalent to changing the `key` prop -
it unmounts and remounts the component instead of preserving the instance.

The method would receive `(nextProps)` as an argument. Alternatively it could be a static method
that receives`(nextProps, prevProps)`.

# Basic example

```diff
class ImgZoomer extends Component<{ imgUrl: string }> {
  constructor(props) {
    super(props);
    this.state = {
      zoom: 1,
      data: null,
      loading: true,
      error: null,
    };
  }
  
  componentDidMount() {
    fetch(this.props.imgUrl).then(
      data => this.setState({data, loading: false}),
      error => this.setState({error, loading: false})
    );
  }
    
+  shouldComponentRemount(newProps) {
+    if (newProps.imgUrl !== this.props.imgUrl) return true;
+  }

  zoomIn = () => this.setState(state => ({ zoom: state.zoom + 1 }));
  
  zoomOut = () => this.setState(state => ({ zoom: state.zoom - 1 }));

  render() {
    if (this.props.error) return 'Error';
    if (this.props.loading) return 'Loading...';
    return (
      <div>
        <ImgComponent scale={(this.state.zoom * 100) + '%'} data={this.state.data} />
        <button onClick={this.zoomIn}>+</button>
        <button onClick={this.zoomOut}>-</button>
      </div>
    );
  }
}
```

# Motivation

The goal is to provide a way for components to manage the relationship between entitites provided by props and
the UI state associated with them.

Example: The [You Might Not Need Derived State][ymnnds-key] blog post recommends a pattern
using an uncontrolled, stateful component with a `key` prop to reset state.

One pitfall with the key approach is that the consumer of the component is responsible for picking a
string for the key that can correctly represent the reset condition, unless the author of the component
writes additional code to make it resilient to props changes that are equivalent to state resets,
similar to ["alternative 1"][ymnnds-alt].

# Detailed design

(TBD if this is feasible)

Lifecycle order:
Not called on initial mount.
Called before `shouldComponentUpdate` otherwise.

# Drawbacks

_TBD_

Why should we *not* do this? Please consider:

- implementation cost, both in term of code size and complexity
- whether the proposed feature can be implemented in user space
- the impact on teaching people React
- integration of this feature with other existing and planned features
- cost of migrating existing React applications (is it a breaking change?)

There are tradeoffs to choosing any path. Attempt to identify them here.

# Alternatives

- Instead of returning true/false, the method could return a string which would
take the place of the `key` prop. Precedence if `key` is also defined TBD.
(In this case maybe it should be called `getDerivedKey`).

# Adoption strategy

If we implement this proposal, how will existing React developers adopt it? Is
this a breaking change? Can we write a codemod? Should we coordinate with
other projects or libraries?

- Not a breaking change

# How we teach this

What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing React patterns?

Would the acceptance of this proposal mean the React documentation must be
re-organized or altered? Does it change how React is taught to new developers
at any level?

How should this feature be taught to existing React developers?

# Unresolved questions

Optional, but suggested for first drafts. What parts of the design are still
TBD?

[ymnnds-key]: https://reactjs.org/blog/2018/06/07/you-probably-dont-need-derived-state.html#recommendation-fully-uncontrolled-component-with-a-key
[ymnnds-alt]: https://reactjs.org/blog/2018/06/07/you-probably-dont-need-derived-state.html#alternative-1-reset-uncontrolled-component-with-an-id-prop
