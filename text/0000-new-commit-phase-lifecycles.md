- Start Date: 2018-03-10
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Add new "commit" phase lifecycle, `getSnapshotBeforeUpdate`, that gets called _before_ mutations are made. Any value returned by this lifecycle will be passed as a third parameter to `componentDidUpdate`.

This lifecycle will be important for [async rendering](https://reactjs.org/blog/2018/03/01/sneak-peek-beyond-react-16.html), where there may be delays between "render" phase lifecycles (e.g. `componentWillUpdate` and `render`) and "commit" phase lifecycles (e.g. `componentDidUpdate`).

# Basic example

Consider the use-case of preserving scroll position within a list as its contents are updated. The way this is typically done is to read `scrollHeight` during render (`componentWillUpdate`) and then adjust it after the update has been committed (`componentDidUpdate`).

Unfortunately this approach **does not work with async rendering**, because there might be a delay between these lifecycles during which the user continues scrolling. The only way to ensure an accurate scroll position was read would be to _force a synchronous render_.

The solution is to introduce a new lifecycle that gets called during the commit phase _before_ mutations have been made to e.g. the DOM. For example:

```js
type Snapshot = number;

class ScrollingList extends React.Component<Props, State> {
  listRef = React.createRef();

  getSnapshotBeforeUpdate(
    prevProps: Props,
    prevState: State
  ): Snapshot | null {
    // Are we adding new items to the list?
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
    // If we have a snapshot value, then we've just added new items.
    // Adjust scroll so these new items don't push the old ones out of view.
    if (snapshot !== null) {
      this.listRef.value.scrollTop +=
        this.listRef.value.scrollHeight - snapshot;
    }
  }

  render() {
    return (
      <div ref={this.listRef}>{/* ...contents... */}</div>
    );
  }
}
```

# Motivation

This lifecycle provides a way for asynchronously rendered components to accurately read values from the host envirnment (e.g. the DOM) before it is mutated.

The [example above](#basic-example) describes one use case in which this could be useful. Others might involve text selection and cursor position, audio/video playback position, etc.

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