- Start Date: 2018-12-10
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Custom Host Node

I was thinking about the use case for useMutationEffect. The use case is to implement your own manual updates to a DOM node, or to a DOM node via a legacy imperative API like jQuery.

The idea is that it is better to do before useLayoutEffect and componentDidMount because they might read layout + followed by setState. So if the first sibling reads layout, the second mutates the DOM, the third reads layout, you get two layout passes instead of one.

However, the problem with useMutationEffect is that refs haven't been swapped yet. During initial render, you don't have access to the new element that you want to mutate. During updates, you have access to the old element but not the new one if a ref has swapped instance.

Another issue with the useMutationEffect API is that normally React does something special during a newly added tree. Instead of performing mutations in the commit phase, we do all the initialization and mutation during the render phase. Only updates get applied.

Another quirk is that these can never really be renderer agnostic since the refs resolve to different things depending on where you render them. Additionally, some environments don't even have mutation APIs like React Fabric. Instead it has a clone API.

So maybe the API should be renderer specific and designed around this exact use case.

## Alternative #1: Hook API

```js
import {useDOMNode} from "react-dom";

function Component(props) {
  let manualChildElement = useDOMNode(() => {
    // create
    let node = document.createElement('div');
    node.style.color = props.color;
    return node;
  }, node => {
    // update
    node.style.color = props.color;
  }, node => {
    // clean up
  });

  return <div>{manualChildElement}</div>;
}
```

## Alternative #2: HoC API

```js
import {createManualDOMNode} from "react-dom";
let CustomDOMNode = createManualDOMNode(props => {
  // create
  let node = document.createElement('div');
  node.style.color = props.color;
  return node;
}, (props, node) => {
  // update
  node.style.color = props.color;
}, (props, node) => {
  // clean up
});

function Component(props) {
  return <div><CustomDOMNode color={props.color} /></div>;
}
```


