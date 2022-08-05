- Start Date: 2022-08-05
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

A way to support slots pattern in React, works similar to [Slots for Web Components](https://developer.mozilla.org/en-US/docs/Web/Web_Components/Using_templates_and_slots)
but more powerful as we can interpolate slots.

# Basic example

In this example we introduced two method called `createHost` and `createSlot`.

`createSlot` creates a component won't be rendered to real DOM but collect props.

`createHost` mounts the children included slotted components and read the
collected slots props, and render the result to real DOM conditionally.

```jsx
import { createHost, createSlot } from "react-create-slots";

const Tag = (props) => <span {...props} />;

const TextField = (props) => {
  const id = useId();

  return createHost(props.children, (Slots) => {
    const labelProps = Slots.get("label");
    const inputProps = Slots.get("input");
    const tagPropsList = Slots.getAll("tag");

    return (
      <div>
        {labelProps && <label htmlFor={inputProps.id ?? id} {...labelProps} />}
        <input id={id} {...label} />
        {tagPropsList.length > 0 && (
          <div>
            {tagPropsList.map((tagProps, index) => (
              <Tag data-index={index} {...tagProps} />
            ))}
          </div>
        )}
      </div>
    );
  });
};

const TextFieldLabel = createSlot("label");
const TextFieldInput = createSlot("input");
const TextFieldTag = createSlot("tag");

export default function App() {
  return (
    <div>
      <TextField>
        <TextFieldInput id="input-id" />
        <TextFieldLabel>It will be rendered above input</TextFieldLabel>
        <TextFieldTag>Tag 1</TextFieldTag>
        <TextFieldTag>Tag 2</TextFieldTag>
      </TextField>
    </div>
  );
}
```

# Motivation

## Composite component in a configure manor

When creating UI library, we have different approaches on the component
API design. The most common way is the Configuration pattern used by
most UI libraries, everything in one component and provides child props
for composition, like `<TextField label helperText />`. It's easy to use,
but when we need to to add some extra props to label like `data-testid`,
then we have to introduce new props for that which will bloat the api
very easily.

Another way to solve this problem is Composition pattern, like
`<TextField><TextFieldLabel /><TextFieldInput /></TextField>`,
it provides the best flexibility, we are free to customise every
part of our component, but it causes another problem: consistency,
we have to organise your sub components exactly same order as expected,
and the biggest problem is that it's harder to communicate between
parent and children.

For a dedicated Design System, we need consistent ui regardless how
we compose it.

Here the Slots pattern solve those problem perfectly, we can compose
our component with both flexibility and consistency, and it's extreme
easy to add A11y support thanks the ability of Inversion of Control.

## Accessible List

Quoted from [the comment](https://github.com/facebook/react/issues/24979#issuecomment-1193176328)
by @devongovett

> A bunch of libraries have a problem where they need to know about certain types of descendants. For example, a list component with keyboard navigation needs to know what elements exist in the collection in order to implement things like typeahead, arrow keys, selection, etc. Reach UI has a good [overview](https://github.com/reach/reach-ui/tree/dev/packages/descendants) of a bunch of different approaches to this. The most commonly used of them involve rendering all of the items to the DOM, and using some kind of context-based registration system to tell the parent about themselves, and the DOM to sort them into the correct order.  
> This has the downside that all of the items must be in the DOM at all times. In some cases, like virtualized scrolling, or a combobox/select where users can set the item without showing the list, some or all of the items shouldn't be rendered to the DOM. In React Aria, we walk the JSX tree to do this, which makes for a more natural API than giving up JSX completely (info). But this breaks composition, because only certain known element types are allowed.

Even with the Reach UI's solution, it doesn't work well with SSR.
With the Slots pattern, we don't render the children to real DOM but
only collect information, so virtualisation is supported by nature.

## Why we want it built in core

I've researched a lot different approaches, none of the current solutions
work perfectly without any drawbacks, my [solution](https://github.com/nihgwu/create-slots)
is the closest one. But as @devongovett pointed out [here](https://github.com/facebook/react/issues/24979#issuecomment-1205909188), `react-reconciler` is designed to do this job,
It would be nice to see it in core, like `react-call-return`.

# Detailed design

The api is very similar to the [deleted](https://github.com/facebook/react/pull/12820)
experimental package [`react-call-return`](https://github.com/facebook/react/pull/11364),
but with a simpler mental model.

`createSlot(slotName)` creates a slot component and won't be rendered to
real DOM but only collect props, which will be used by `createHost`

`createHost(children, callback)` mounts the children included slotted
components and read the collected slots' props, and render the result to
real DOM conditionally. The argument of `callback` provides the following
methods:

- `get(slotName: string)` returns the registered slot's props by name,
  will return the last registered named slot if the host expect only one
  but consumer provides more.
- `getAll(slotName: string)` returns all the registered slots as array
  this method is useful to create a list.
- `getChildren()` returns the rest of children without slots, e.g.
  `<Host>rest<Slot /></Host>` it will return `rest`.

# Drawbacks

Even we are going to support this feature in a separate package or entry,
it will still increase the bundle size of React a bit as we need core support.

# Alternatives

- Bring back `react-call-return` which also could be used to implement this
  feature, but the mental model is hard to understand.
- Leave it to userspace to implement with current api, like [create-slots](https://github.com/nihgwu/create-slots), but not very efficient and have
  some drawbacks, e.g. unable to catch children not wrapped in slots.

# Adoption strategy

There is no breaking changes, it's a new feature more for UI library authors
instead of application developers.

# How we teach this

Slots pattern is a native feature of Web Components, other popular frameworks
like Vue and Svelte also provide similar concepts. So the concept itself is
easy to understand, we only have to document the api.

# Unresolved questions

- Finalise the namings
- Do we need to provide internal key for list rendering, though I don't see it
  in `react-call-return`
- Adding to a new package or to `React` directly
