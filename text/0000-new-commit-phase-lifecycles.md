- Start Date: 2018-03-10
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Add new "commit" phase lifecycle, `getSnapshotBeforeUpdate`, that gets called _before_ mutations are made. Any value returned by this lifecycle will be passed as the third parameter to `componentDidUpdate`.

This lifecycle is important for [async rendering](https://reactjs.org/blog/2018/03/01/sneak-peek-beyond-react-16.html), where there may be delays between "render" phase lifecycles (e.g. `componentWillUpdate` and `render`) and "commit" phase lifecycles (e.g. `componentDidUpdate`).

# Basic example

Consider the use case of preserving scroll position within a list as its contents are updated. The way this is typically done is to read `scrollHeight` during render (`componentWillUpdate`) and then adjust it after the update has been committed (`componentDidUpdate`).

Unfortunately this approach **does not work with async rendering**, because there might be a delay between these lifecycles during which the user continues scrolling. The only way to ensure an accurate scroll position is read would be to _force a synchronous render_.

The solution is to introduce a new lifecycle that gets called during the commit phase before mutations have been made to e.g. the DOM. For example:

```js
type Snapshot = number | null;

class ScrollingList extends React.Component<Props, State, Snapshot> {
  listRef = React.createRef();

  getSnapshotBeforeUpdate(
    prevProps: Props,
    prevState: State
  ): Snapshot {
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
    snapshot: Snapshot
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

This lifecycle provides a way for asynchronously rendered components to accurately read values from the host environment (e.g. the DOM) before it is mutated.

The [example above](#basic-example) describes one use case in which this could be useful. Others might involve text selection and cursor position, audio/video playback position, etc.

# Detailed design

Add a new effect type, `Snapshot`, and update `ReactFiberClassComponent` to assign this type when updating components that define the new `getSnapshotBeforeUpdate` lifecycle.

During the  `commitAllHostEffects` traversal, call `getSnapshotBeforeUpdate` for any fiber tagged with the new `Snapshot` effect type. Store return value on the instance (as `__reactInternalSnapshotBeforeUpdate`) and later pass to `componentDidUpdate` during `commitLifeCycles`.

### New DEV warnings

Add DEV warnings for the following conditions:
* Undefined return values for `getSnapshotBeforeUpdate`
* Components that define `getSnapshotBeforeUpdate` without also defining `componentDidUpdate

### Flow

Flow will also need to be updated to add a third `Snapshot` type parameter to `React.Component` to ensure consistency for the return type fo `getSnapshotBeforeUpdate` adn the new parameter passed to `componentDidUpdate`. This new type parameter will be declared like so:

```js
// If there is a State type:
class Example extends React.Component<Props, State, Snapshot> {}

// If there is no State type:
class Example extends React.Component<Props, State = void, Snapshot> {}
```

# Drawbacks

Each new lifecycle adds complexity and makes the component API harder for beginners to understand. Although this lifecycle _is important_, it will probably _not be used often_, and so I think the impact is minimal.

# Alternatives

A new commit-phase lifecycle is necessary. The signature does not have to match the one proposed by this RFC however. Below are some alternatives that were considered.

### Static method

The most recently-added lifecycle, `getDerivedStateFromProps`, was a static method in order to prevent unsafe access of instance properties. That concern is less relevant in this case though, because this lifecycle is called during the commit phase.

```js
class ScrollingList extends React.Component<Props, State> {
  state = {
    listHasGrown: false,
    listRef: React.createRef(),
    prevList: this.props.list
  };

  static getDerivedStateFromProps(
    nextProps: Props,
    prevState: State
  ): $Shape<State> | null {
    if (nextProps.list !== prevState.prevList) {
      return {
        listHasGrown:
          nextProps.list.length > prevState.prevList.length,
        prevList: nextProps.list
      };
    } else if (prevState.listHasGrown) {
      return {
        listHasGrown: false
      };
    }

    return null;
  }

  static getSnapshotBeforeUpdate(
    prevProps: Props,
    prevState: State
  ): Snapshot | null {
    if (prevState.listHasGrown) {
      return prevState.listRef.value.scrollHeight;
    }

    return null;
  }

  // ...
}
```

This approach was not chosen because of the added complexity of storing additional values (including refs) in `state`.

### No return value

The proposed lifecycle will be the first commit phase lifecycle with a meaningful return value and the first lifecycle whose return value is passed as a parameter to another lifecycle. Likewise, the new parameter for `componentDidUpdate` will be the first passed to a lifecycle that isn't some form of `Props` or `State`. This adds some complexity to the API, since it requires a more nuanced understanding the relationship between `getSnapshotBeforeUpdate` and `componentDidUpdate`.

An alternative would be to scrap the return value in favor of storing snapshot values on the instance. This has the added benefit of not requiring any changes to be made to Flow.

```js
class ScrollingList extends React.Component<Props, State> {
  listRef = React.createRef();
  listScrollHeight = null;

  getSnapshotBeforeUpdate(
    prevProps: Props,
    prevState: State
  ) {
    if (prevProps.list.length < this.props.list.length) {
      this.listScrollHeight = this.listRef.value.scrollHeight;
    }
  }

  componentDidUpdate(prevProps: Props, prevState: State) {
    if (this.listScrollHeight !== null) {
      this.listRef.value.scrollTop +=
        this.listRef.value.scrollHeight - snapshot;
      this.listScrollHeight = null;
    }
  }

  // ...
}
```

Ultimately, the team voted against this approach because it encourages mutations and may invite other side-effects in a lifecycle that is intended to be used for a very specific purpose.

# Adoption strategy

Since this lifecycle- and async rendering in general- is new functionality, adoption will be organic. Documentation and dev-mode warnings have already been created to encourage people to move away from render phase lifecycles like `componentWillUpdate` in favor of commit phase lifecycles.

# How we teach this

Lifecycle documentation on the website. Add a before an after example (like [the one above](#basic-example)) to the [Update on Async Rendering](https://github.com/reactjs/reactjs.org/pull/596) blog post "recipes".

# Unresolved questions

None presently.