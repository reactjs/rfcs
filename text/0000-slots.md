- Start Date: 2022-08-05
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

A way to support slots pattern in React, works similar to [Slots for Web Components](https://developer.mozilla.org/en-US/docs/Web/Web_Components/Using_templates_and_slots) but more powerful as we can interpolate slots.

# Basic example

In this example we introduced two methods called `createHost` and `createSlot`.

`createSlot` creates a component which won't be rendered to real DOM but only collect node info.

`createHost` mounts the children which hosting slotted components and read the collected slots info, and render the result to real DOM conditionally.

```jsx
import { createHost, createSlot } from "react-create-slots";

const TextFieldLabel = createSlot();
const TextFieldInput = createSlot();
const TextFieldTag = createSlot();

const Tag = (props) => <span {...props} />;

const TextField = (props) => {
  const id = useId();

  return createHost(props.children, (slots) => {
    // library author can create utils to reduce the boilerplate
    const labelSlot = slots.findLast((slot) => slot.type === TextFieldLabel);
    const inputSlot = slots.findLast((slot) => slot.type === TextFieldInput);
    const tagSlots = slots.filter((slot) => slot.type === TextFieldTag);

    return (
      <div>
        {labelSlot && (
          <label htmlFor={inputSlot?.props.id ?? id} {...labelSlot.props} />
        )}
        <input id={id} {...inputSlot?.props} />
        {tagSlots.length > 0 && (
          <div>
            {tagSlots.map((tagSlot, index) => (
              <Tag key={index} data-index={index} {...tagSlot.props} />
            ))}
          </div>
        )}
      </div>
    );
  });
};

export default function App() {
  return (
    <div>
      <TextField>
        <TextFieldInput id="input-id" />
        <TextFieldLabel>I will be rendered before input</TextFieldLabel>
        <TextFieldTag>Tag 1</TextFieldTag>
        <TextFieldTag>Tag 2</TextFieldTag>
      </TextField>
    </div>
  );
}
```

# Motivation

## Composite component in a configure manner

When creating UI library, we have different approaches on the component API design. The most common way is the Configuration pattern used by most UI libraries, everything in one component and provides child props for composition, like `<TextField label helperText />`. It's easy to use, but when we need to to add some extra props to label like `data-testid`, then we have to introduce new props for that which will bloat the api very easily.

Another way to solve this problem is Composition pattern, like `<TextField><TextFieldLabel /><TextFieldInput /></TextField>`, it provides the best flexibility, we are free to customise every part of our component, but it causes another problem: consistency, we have to organise your sub components exactly the same order as expected, and the biggest problem is that it's very hard and cumbersome to communicate between parent and children.

For a dedicated Design System, we need consistent ui regardless how we compose it. Here the Slots pattern solve those problem perfectly, we can compose our component with both flexibility and consistency, and it's extreme easy to add A11y support thanks the ability of Inversion of Control. We don't need to using React Context to communicate between parent and children, and no rerendering needed to sync the data.

## Accessible List, virtualisation and custom renderer

Quoted from [the comment](https://github.com/facebook/react/issues/24979#issuecomment-1193176328) by @devongovett

> A bunch of libraries have a problem where they need to know about certain types of descendants. For example, a list component with keyboard navigation needs to know what elements exist in the collection in order to implement things like typeahead, arrow keys, selection, etc. Reach UI has a good [overview](https://github.com/reach/reach-ui/tree/dev/packages/descendants) of a bunch of different approaches to this. The most commonly used of them involve rendering all of the items to the DOM, and using some kind of context-based registration system to tell the parent about themselves, and the DOM to sort them into the correct order.  
> This has the downside that all of the items must be in the DOM at all times. In some cases, like virtualized scrolling, or a combobox/select where users can set the item without showing the list, some or all of the items shouldn't be rendered to the DOM. In React Aria, we walk the JSX tree to do this, which makes for a more natural API than giving up JSX completely (info). But this breaks composition, because only certain known element types are allowed.

Even with the Reach UI's solution, it doesn't work well with SSR. With the Slots pattern, we don't render the children to real DOM but only collect information, so virtualisation is supported by nature.

Also the Slots proposal actually enabled another way of creating custom renderer, we don't need to import the `react-reconciler` which added a lot of bundle size, instead we use the version already bundled in `react-dom` or `react-native`, we define custom tags by `createSlot` and implement custom rendering in `createHost`, an easy showcase is implementing dynamic and extendable declarative configuration.

## Why we want it built in core

I've researched a lot different approaches, none of the current solutions work perfectly without any drawbacks, my [solution](https://github.com/nihgwu/create-slots) is the closest one, but as @devongovett pointed out [here](https://github.com/facebook/react/issues/24979#issuecomment-1205909188), `react-reconciler` is designed to do this job, It would be better to have it in core.

It seems `React.Children.forEach` can do the same job demoed in the example above, but there are a lot limitations which are well documented [here](https://github.com/reach/reach-ui/tree/dev/packages/descendants), with this proposal we are free to extend the slot and compose in any way, e.g.

```jsx
const StyledLabel = styled(TextField)``;

const PartialTextField = ({ show }) => (
  <>
    {show && <StyledLabel>Dynamic styled label</StyledLabel>}
    <>
      <TextFieldInput />
    </>
  </>
);

export default function App() {
  return (
    <TextField>
      <PartialTextField />
    </TextField>
  );
}
```

# Detailed design

The api is very similar to the [deleted](https://github.com/facebook/react/pull/12820) experimental package [`react-call-return`](https://github.com/facebook/react/pull/11364), but with a simpler mental model.

`createSlot<T extends React.ElementType>(Fallback?: T)` creates a slot component and won't be rendered to real DOM but only collect node info, which will be used by `createHost`. `Fallback` is a fallback component that if the slot is used without HostSlots(outside of `createHost`), it will be fallback to a normal component, which is similar to `slot` property for Web Components, that if the slot is not used in template it will act as a normal component. If `Fallback` is not provided, it will create a pure slot component that nothing will be rendered if it's used without HostSlots. (Not sure about the `Fallback` argument as it's not able to implement with `react-call-return`.)

`createHost(children: React.ReactNode, callback: (slots: React.ReactElement[]) => JSX.Element | null)` mounts the children which hosting slotted components and return the collected slots elements in callback, and then we can render the result to real DOM conditionally. We can find the slots by filter the `slots`(shown in the example above).

For `children` of `createHost`, similar to `react-call-return`, slot component can be extended, wrapped with `Fragment`, but can't be wrapped with Host components(like `div`), see the following use cases

```jsx
const Slot = createSlot();
const Host = (props) => createHost(props.children, (slots) => slots);

// valid but nothing will be rendered for `Slot` as it's not wrapped with `Host`
const Comp = () => (
  <div>
    <Slot />
  </div>
);

// valid extension
const ExtendedSlot = (props) => (
  <>
    <Slot prop1="props1" {...props} />
  </>
);

// invalid extension
const ExtendedSlot = (props) => (
  <div>
    <Slot {...props} />
  </div>
);

// valid composition
const Comp1 = () => (
  <Host>
    <>
      <Slot />
      <ExtendedSlot />
    </>
  </Host>
);

// invalid composition
const Comp1 = () => (
  <Host>
    <div>
      <Slot />
    </div>
  </Host>
);
```

For nested slots, `createHost` works like peeling the onion, it will only collect the top level slots, nested slots will be handled at next tick if they are included in `callback`'s return, to continue with the very first example:

```jsx
const NestedField = createSlot();

const NestedControl = (props) =>
  createHost(props.children, (slots) => {
    const textFieldSlot = slots.findLast((slot) => slot.type === NestedField);
    const labelSlot = slots.findLast((slot) => slot.type === TextFieldLabel);

    return (
      <div>
        {labelSlot && (
          <label
            style={{ fontWeight: "bold", color: "red" }}
            {...labelSlot.props}
          />
        )}
        {textFieldSlot && <TextField {...textFieldSlot.props} />}
      </div>
    );
  });

export default function App() {
  return (
    <div>
      <NestedControl>
        <TextFieldLabel>I have different style</TextFieldLabel>
        <NestedField>
          <TextFieldInput />
          <TextFieldLabel>Nested field</TextFieldLabel>
        </NestedField>
      </NestedControl>
    </div>
  );
}
```

If we support `Fallback` in `createSlot`, then for `const TextFieldLabel = createSlot('label')`, `<TextFieldLabel>Label</TextFieldLabel>` outside of `TextField` will be rendered as `<label>Label</label>`, and in `createHost`'s `callback`, we can return the slot elements directly as after the the onion been peeled, they won't be caught by `createHost` again, so they will fallback to normal components, e.g.

```jsx
const Header = createSlot("header");
const Body = createSlot("main");
const Footer = createSlot("footer");

const Layout = (props) =>
  createHost(props.children, (slots) => {
    const header = slots.find((slot) => slot.type === Header);
    const body = slots.find((slot) => slot.type === Body);
    const footer = slots.find((slot) => slot.type === Footer);

    return (
      <div>
        {header}
        <div>
          {body}
          {footer}
        </div>
      </div>
    );
  });

export default function App() {
  return (
    <Layout>
      <Header>Header</Header>
      <Body>Body</Body>
    </Layout>
  );
}
// will produce the following result:
// <div>
//   <header>Header</header>
//   <div>
//     <main>Body</main>
//   </div>
// </div>
```

# Drawbacks

- Even we are going to support this feature in a separate package or entry, it will still increase the bundle size of React a bit as we need support from core.
- New concepts and new apis will increase the complexity of learning

# Alternatives

- Bring back `react-call-return` which also could be used to implement this feature [demo](https://codesandbox.io/s/long-hill-os6msf?file=/src/App.js), but the mental model is hard to understand.
- Leave it to user space to implement with current api, like [create-slots](https://github.com/nihgwu/create-slots), but not very efficient and have some drawbacks, e.g. unable to catch children not wrapped in slots, for list slots we have to force update to get the correct index.

# Adoption strategy

There is no breaking changes, it's a new feature more for UI library authors instead of application developers.

# How we teach this

Slots pattern is a native feature of Web Components, other popular frameworks like Vue and Svelte also provide similar concepts. So the concept itself is easy to understand, we only have to document the api.

# Unresolved questions

- Finalise the namings
- Do we need to provide internal key for list rendering?
- Adding to a new package or new entry or exposing from `React` directly?
- Should we support `Fallback` for `createSlot`?
- Do we allow arbitrary content which is not wrapped in slot components? It's supported by Web Components(default slot `<slot></slot>`), but disallowed in `react-call-return`
