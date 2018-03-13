- Start Date: 2018-03-10
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Add new "commit" phase lifecycle, `getSnapshotBeforeUpdate`, that fires during an update, before mutations are made. The return value from this lifecycle will be passed as a third parameter to `componentDidUpdate`.

This lifecycle will be important for [async rendering](https://reactjs.org/blog/2018/03/01/sneak-peek-beyond-react-16.html), where there may be delays between "render" phase lifecycles (e.g. `componentWillUpdate` and `render`) and "commit" phase lifecycles (e.g. `componentDidUpdate`).

# Basic example

Consider the use-case of preserving scroll position within a list as its contents are updated. The way this is typically done is to read `scrollHeight` during render (using `componentWillUpdate`) and then adjust it after the update has been committed (using `componentDidUpdate`).

This approach does not work with async rendering, since there might be a delay between these lifecycles during which the user might continue scrolling. The only way to safely do this with current lifecycles would be to _force a synchronous render_.

The solution to this is to introduce a new lifecycle that gets called during the commit phase _before_ mutations have been made to e.g. the DOM. For example:

```js
type Snapshot = number;

class ScrollingList extends React.Component {
  listRef = React.createRef();

  getSnapshotBeforeUpdate(prevProps: Props, prevState: State): Snapshot | null {
    // We are adding new items to the list.
    // Capture the current height of the list so we can adjust scroll later.
    if (prevProps.list.length < this.props.list.length) {
      return this.listRef.value.scrollHeight;
    }

    return null;
  }

  componentDidUpdate(
    prevProps: Props,
    prevState: State,
    snapshot: Snapshot | null
  ) {
    // If we have an snapshot then we've just added new items.
    // Adjust scroll so these new items don't push the old ones out of view.
    if (snapshot !== null) {
      this.listRef.value.scrollTop +=
        this.listRef.value.scrollHeight - snapshot;
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