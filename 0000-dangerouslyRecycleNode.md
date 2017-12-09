- Start Date: 2017-12-09
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Allow a DOM node from an unmounted component to be reused within a newly mounted component. This is particularly helpful when doing cross-route transition animations, or forwarding video playback to a detailed view from a feed.

This is a new feature request, and would have no backwards incompatibility issues.

# Basic example

The proposed solution is to add a `dangerouslyRecycleNode={id}` react internal prop that can be added to raw dom nodes (like `dangerouslySetInnerHTML`). If an unmounting node contains the prop, React caches the node and if a mounting node has the same `id`, it will reuse the exact DOM node from the previously unmounted component.

```js
class FeedPage extends React.Component {
  changeRoute(id) {
    this.history.push(`/feedItem/${id}/`);
  }
  render() {
    const { feedItems } = props;
    return (
      <div>
        {feedItems.map(({ imageSrc, id }) =>
          <img
            class="small-img"
            key={id}
            src={imageSrc}
            onClick={this.changeRoute.bind(this, id)}
            dangerouslyRecycleNode={id}
          />)}
      </div>
    );
  }
}
class DetailPage extends React.Component {
  render() {
    const { imageSrc, id } = this.props.feedItem;
    return (
      <img
        class="large-img"
        src={imageSrc}
        dangerouslyRecycleNode={id}
      />
    );
  }
}
```

# Motivation

Reusing DOM nodes between routes can be a wonderful user experience, but is impossible within React core currently. The library [react-overdrive](https://github.com/berzniz/react-overdrive) was the inspiration for the API of this RFC. This would be a fairly simple addition to React core that would allow for some beautifully declarative route transitions.

## Detailed Design

The proposal is to add a `dangerouslyRecycleNode` prop to raw DOM nodes. Internally on unmount if a node has the prop set, it will add it to the internal `domNodeCache[key]`. If another node with the matching prop + recycle id is then mounted, React will not create a new node, but will extract the cached DOM node and use it in the newly mounted component. It will then immediately remove it from cache.

We will log an error to console (in dev mode) after 30s of a node being added to the cache if it is not reused.

## Drawbacks

The main drawback is around the lifecycle of the cached node from the unmounting component. This could potentially cause memory leaks.

The proposed prop API is named `dangerouslyRecycleNode` because one option is to keep it cached until a node with the matching `id` is rendered (which may never happen). Another option would be to keep the reference in cache for a certain amount of time, and since this will mostly be used for reuse between routes, that should not be too long. Unless you're codesplitting by route and have not preloaded the route you're going to next. That is why a cache expiration timeout is tricky. My preference is to keep it around until it is reused, and to keep the name as `dangerous` so people are aware that they can be causing memory leaks if they don't actually reuse the node anywhere.

## Alternatives
Unsure.

## Adoption strategy
This would be a new internal prop on raw DOM nodes. People can adopt however they want with no chance of backwards incompatibilities.

## How we teach this
Docs

## Unresolved questions
Cache expiration for recyclable nodes.