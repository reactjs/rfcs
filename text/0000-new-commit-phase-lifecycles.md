- Start Date: 2018-03-10
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Add two new "commit" phase lifecycles, `componentIsMounting` and `componentIsUpdating`, that fire before mutations are made.

These will be important for [async rendering](https://reactjs.org/blog/2018/03/01/sneak-peek-beyond-react-16.html), where there may be delays between "render" phase lifecycles (e.g. `componentWillUpdate` and `render`) and "commit" phase lifecycles (e.g. `componentDidUpdate`).

# Basic example

Consider the use-case of preserving scroll position within a list as its contents are updated. The way this is typically done is to read `scrollHeight` during render (using `componentWillUpdate`) and then adjust it after the update has been committed (using `componentDidUpdate`).

This approach does not work with async rendering, since there might be a delay between these lifecycles during which the user might continue scrolling. The only way to safely do this with current lifecycles would be to _force a synchronous render_.

The solution to this is to introduce a new lifecycle that gets called during the commit phase _before_ mutations have been made to e.g. the DOM. For example:

```js
class ScrollingList extends React.Component {
  listRef = React.createRef();
  cachedScrollHeight = null;

  componentIsUpdating(prevProps, prevState) {
    // We are adding new items to the list.
    // Store the current height of the list so we can compare after updating,
    // And adjust scroll so that new comments don't push old ones out of view.
    if (this.props.list.length > prevProps.list.length) {
      this.cachedScrollHeight = this.listRef.value.scrollHeight;
    }
  }

  componentDidUpdate(prevProps, prevState) {
    // If we have a cached scroll height then we've just added new items.
    // Adjust scroll so these new items don't push the old ones out of view.
    if (this.cachedScrollHeight !== null) {
      this.listRef.value.scrollTop +=
        this.listRef.value.scrollHeight - this.cachedScrollHeight;
      this.cachedScrollHeight = null;
    }
  }

  render() {
    return <div ref={this.listRef}>{/* ...contents... */}</div>;
  }
}
```

# Motivation

Coming soon...

# Detailed design

Coming soon...

# Drawbacks

Coming soon...

# Alternatives

Coming soon...

# Adoption strategy

Coming soon...

# How we teach this

Coming soon...

# Unresolved questions

Coming soon...