- Start Date: 2023-10-10
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

This RFC proposes adding a utility to dispatch custom events in components. The RFC only includes a proposal for a utility function called `dispatchCustomEvent`: a utility that introduces some standards for emitting custom events from components.

# Basic example

A simple example of a component emitting a custom event

```tsx
import { dispatchCustomEvent } from "react"; // 🤞🏼

const MyComponent = ({ onChange }) => {
  React.useEffect(() => {
    subscribeToSomeUserEvent((someArbitraryData) => {
      dispatchCustomEvent(onChange, {
        type: "change",
        detail: { value: someArbitraryData },
      });
    });
  }, []);
  return <>{/*irrelevant*/}</>;
};
```

A simple example of consuming the data of the event

```tsx
<MyComponent
  onChange={(evt) => {
    console.log(evt.detail.value);
  }}
/>
```

# Motivation

<!-- Why are we doing this? What use cases does it support? What is the expected
 outcome? -->

This idea came to me after working in a component library for a couple of years and observing the state of widely used component libraries in open source (such as material-ui).

The issue is multi-faceted.

1. The lack of a standard API for custom events leads to inconsistent and unpredictable APIs exposed by library developers. The larger the library, the more likely it is that some inconsistency will manifest - since larger libraries tend to have more developers, with different development styles.
2. The lack of a standard API for custom events reduces interoperability of utility component or libraries that act on component/element events.

### More info on inconsistent custom event APIs in the wild

From my experience working in a component library, and observing popular open source libraries such as material-ui (and others), I've observed the following types be used for custom event handlers. This adds some friction when developing libraries that act on events.

```ts
type SlewOfCustomEventHandlers =
  | (data: ArbitraryType) => void
  | (data: ArbitraryType, originalEvt: React.SyntheticEvent) => void
  | (evt: React.SyntheticEvent, data: ArbitraryType) => void
  | (evt: React.SyntheticEvent) => void
  | (evt: Omit<React.ChangeEvent, 'target'> & { target: { value: ArbitraryType } }) => void
  | (params: { data: ArbitraryType, originalEvt: React.SyntheticEvent }) => void
  | (params: {data: ArbitraryType} & React.SyntheticEvent) => void
  | () => void
```

> This type shows many of the types of observed but I've ommitted types that include a native DOM event in place of the React.SyntheticEvent, which happen occasionally when components abstract behaviors that are attached to the `window`, `document`, or `body` element.

### Example of interoperability problem

Lets say you have a react library that provides you with an API for adding validation to input fields.

The library may expose a hook that allows you to customize the validations for a given field, and provides you with a prop to monitor changes in the field.

```ts
const propsToMonitorChanges = useValidatedInput({
  validations: {
    required: true,
    email: true,
    // etc
  },
});

const { onChange } = propsToMonitorChanges;

return <input onChange={onChange} />;
```

> NOTE: In this example, I am presuming that the library supports both controlled and uncontrolled modes for input, hence the need for the generated `onChange`. When the value is controlled, the change is monitored using the `value` prop, which should be passed to the hook.

This example would work fine without any issues.

But lets say the `input` field must change from a simple `input`, to a complex field widget, like a datepicker with a pop-up.

```ts
const propsToMonitorChanges = useValidatedInput({
  validations: {
    minYear: 2022,
    // etc
  },
  using: DateValidator,
});

const { onChange } = propsToMonitorChanges;

return <DatePicker onChange={onChange} />;
```

A couple of things will need to change.

1. ✅ the `value` supported by the field may not be a `string` or `number` (as it is in the native input). Maybe a `Date` or `Temporal.PlainDate`
2. ✅ the validators should change to something that can handle the new data type
3. ❌ the `onChange` callback must handle the api exposed by `DatePicker`

At this point, there are a couple of options available to each group of developers (developers of the `useValidatedInput` hook, and consumers of the hook).

1. The developers of the hook can accept a "strategy" function to read values from the `onChange` event

> ❌ Interop exhibit 1. There has to be a strategy that is injected to properly handle the event. This is adding some code complexity

```ts
const propsToMonitorChanges = useValidatedInput({
  getValueFromEvent: (customEvent) => getDateSomeHow(customEvent);
  validations: {
    minYear: 2022,
    // etc
  },
  using: DateValidator
});

const {onChange} = propsToMonitorChanges;

return <DatePicker onChange={onChange} />
```

2. The consumers of the hook can wrap the `onChange` event so that it has the shape of a native event

> ❌ Interop exhibit 2. Users of the validation library have to implement adapters for things to work, adding some code complexity.

```ts
const propsToMonitorChanges = useValidatedInput({
  validations: {
    minYear: 2022,
    // etc
  },
  using: DateValidator,
});

const { onChange } = propsToMonitorChanges;

return (
  <DatePicker
    onChange={(customEvent) => {
      onChange({
        target: {
          value: getDateSomeHow(customEvent),
        },
      });
    }}
  />
);
```

# Detailed design

<-- This is the bulk of the RFC. Explain the design in enough detail for somebody
familiar with React to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here. -->

The proposal is fairly simple: Expose a utility function to emit custom events. Custom events should resemble Web APIs where possible, and deviate if necessary to fit into a component oriented architecture.

## Definitions

> Note the following definitions are meant to give a rough idea of the proposal, and to communicate the core of the proposal. The extended parts of the proposal will include additional types if necessary

Please read the comments explaining the role of each type

```ts
/**
 * Leaning on the web standard for `CustomEvent`s, this is a subset of the native api.
 *
 * A few of the properties are false by default. This is solely to provide interoperability between
 * events, as they will never be true in a component architecture.
 **/
interface CustomEvent<DetailType> {
  readonly type: string;
  readonly detail: DetailType;
  readonly cancelable: boolean;
  readonly timeStamp: number;

  readonly isTrusted: false;
  readonly bubbles: false;
  readonly composed: false;
}

type CustomEventHandler = <DetailType>(evt: CustomEvent<DetailType>) => void;

/**
 * These are the properties needed to create and dispatch a custom event
 */
interface CustomEventInit<DetailType> {
  type: stirng;
  detail?: DetailType;
}

type DispatchCustomEvent = <DetailType>(handler: CustomEventHandler<DetailType>, eventInit: CustomEventInit<DetailType>) =>

/**
 * This utility function can be used when implementing components which emit custom events.
 * It can be used in cases where the custom event serves as an abstraction of multiple different
 * user interactions, or events that are not directly triggered by users but by some external
 * event handled in the  component (such as receiving a new message from a websocket).
 *
 **/
const dispatchCustomEvent: DispatchCustomEvent;
```

## Niche cases 1: Event default behavior

In addition to the core part of the proposal, it would be good to allow library developers to expose a default event behavior that is cancelable.

### Example: Form component with validation behavior on submit

Scenario:

Lets say a library developer implements a generic `Form` component. The `Form` component exposes an `onSubmit` event prop, which is called when the data is submitted. The `Form` component also triggers client-side field validations when the form is submitted.

Problem:

A user of the form library wants to perform some other validations, or some async process before the validations are triggered.

### Proposal

Leaning on the standard event APIs, we can lean on the `preventDefault` method. `dispatchCustomEvent` can accept a `defaultBehavior` option, which can be used at the call site to define the default behavior of the event. Internally, `dispatchCustomEvent` can defer the execution of `defaultBehavior` until after the user-defined event handler is called, allowing the user-defined handler to cancel the default behavior.

#### Types

The following type extensions will be needed

```ts
interface CustomEvent<DetailType> {
  readonly cancelable: boolean;
  readonly preventDefault(): void;
  readonly isDefaultPrevented(): boolean;
  readonly defaultPrevented: boolean;
}

interface CustomEventInit<DetailType> {
  defaultBehavior?(): void;
  cancelable?: boolean;
}
```

#### API 1: Library developer

Library developers can configure a defaultBehavior, which will execute after the provided callback is executed.

```ts
import { dispatchCustomEvent } from "react";

const Form = ({ onSubmit, children }) => {
  return (
    <form
      onSubmit={(evt) => {
        // evt.preventDefault(); // Prevent native browser submit -- unrelated to proposal but necessary in component libraries. Just calling it out here to disambiguate between the two. The original preventDefault is irrelevant for porposes of this proposal

        dispatchCustomEvent(onSubmit, {
          type: "submit",
          cancelable: true,
          defaultBehavior: () => {
            performValidationsOnFields(); // Default behavior that is cancelable
          },
        });
      }}
    >
      {children}
    </form>
  );
};
```

#### API 2: Library consumer

Consumers of the form library can prevent the default behavior with the familiar `preventDefault` method, then wrap the validations behavior however they need.

```ts
<Form
  onSubmit={(evt) => {
    evt.preventDefault(); // Prevents field validations

    someAsyncAction().then(() => {
      someRef.current.performValidations();
    });
  }}
>
  {/* Form fields omitted */}
</Form>
```

## Niche cases 2: Imperative event API

Following the scenario above, notice that in the example where the validation logic is wrapped by the consumer of the `Form` component, I used some arbitrary ref reference (so to not imply that the ref has to come from anywhere in particular). However, it may be practical or beneficial to allow events to expose an imperative API. This can reduce the need for refs in some cases, where you just need to perform some imperative action during the event. For example, in native events, we can do things such as `evt.target.focus()`. Following this, it may be useful to allow exposing an imperative API in custom events, where component developers can expose an abstract API for performing some action in the event.

### Example 1: (Building on the last Form example) Wrapping default validation logic in a form

The scenario for this example builds on top of the scenario for "Niche cases 1: Form component with validation behavior on submit.". As showed in the proposed solution for the problem, the developer has to access some ref to perform the validations after some async process.

### Example 2: Moving focus to some abstracted element

Another example is needing to move focus to some element abstracted by a component.

Scenario:

Building on the Form example, lets say there is a requirement to `focus` on the input with errors when the validations are performed.

Problem:

The `Form` component may already have all the ingredients and information to know which field needs to be focused. It would be practical if it can expose a simple API to focus on the relevant field. Otherwise, it must communicate enough information for the caller to be able to move focus to the field, thus creating the posibility for increased code complexity,or the need to manage more refs.

### Proposal

A simple extension of the core proposal that address this issue is to extend the `CustomEvent` api to support custom imperative APIs. It feels intuitive to build upon the DOM event standard of setting the `target` property.

I can see this is potentially a controversial part of the proposal. (More on this in the Drawbacks section). The thinking behind this choice is the following:

1. Custom events are partly intended to increase component encapsulation, while also maintaining some of the familiar structures established in the DOM. In the DOM, the `target` property is the `EventTarget` that triggered the event. Adopting the models in the DOM, it makes sense to think of the component as the event target of a react custom event. As the event target, the component can be accessible via `event.target`; However, when the paradigm of a component changes from declarative to imperative, we refer to the component as it's exposed `ref`, thus it seems reasonable that `event.target` gives access to the `ref` of the component.

2. This choice can facilitate improving interoperability of libraries, as it makes it trivial for library developers to expose APIs such as `event.target.value` or `event.target.focus()`

#### Types

The follow type extensions will be needed for this part of the proposal

```ts
/**
 * A new generic type is needed
 */
interface CustomEvent<DetailType, TargetType> {
  target: TargetType;
}

/**
 * A new generic type is needed
 */
interface CustomEventInit<DetailType, TargetType> {
  target?: TargetType;
}
```

#### API 1: Library developer exposes event target

For this part of the proposal, there are a couple of choices that can be made depending on the mental model the react team feels is more appropriate to adopt.

The choices for the models that I can identify are the following:

1. The target refers to the component emitting the event.
2. The target refers to some abstract EventTarget that is not inherently coupled to the identity of the component.

Depending on which of these choices are made, the API for configuring the target can be expressed as more "closed" (where some assumptions are made automatically) or "opened" where it is completely arbitrary and up to the developer to make a deliberate choice.

##### Example of "closed" api

In the closed API, some assumptions can be tied to the generation of component refs. The benefit may be along the lines of consistency accross custom event implementations, but the trade-offs will likely be manifested as increased complexity of implementation, or blurring the porpose of the existing APIs.

##### Option 1: `useImperativeHandle`

In this example, `dispatchCustomEvent` is provided by the `useImperativeHandle` and the target is automatically bound.

```tsx

const MyComponent = React.forwardRef(({ onCustomEvent }, ref) => {
  const {dispatchCustomEvent} = useImperativeHandle(ref, () => {
    performSomeComponentAction: () => {}
    /* generate handle*/
  })

  React.useEffect(() => {
    subscribeToSomething(() => {
      dispatchCustomEvent(onCustomEvent, {
        type: 'custom-event'
      })
    })
  }, [])
})


// Consumer
<MyComponent onCustomEvent={(evt) => {
  if (someCondition) {
    evt.target.performSomeComponentAction();
  }
}} />
```

##### Option 2: new hook for defining imperative handle that emits custom events

Its likely cleaner to introduce a new API instead of changing `useImperativeHandle`. The example will look the same except with a different hook name

```tsx
import { useEventTargetImperativeHandle } from "react";

const MyComponent = React.forwardRef(({ onCustomEvent }, ref) => {
  const { dispatchCustomEvent } = useEventTargetImperativeHandle(ref, () => {
    performSomeComponentAction: () => {};
    /* generate handle*/
  });

  React.useEffect(() => {
    subscribeToSomething(() => {
      dispatchCustomEvent(onCustomEvent, {
        type: "custom-event",
      });
    });
  }, []);
});
```

##### Example of "opened" api

An opened API will probably be much simpler to implement in react, but increases the chances of the event target beeing inconsistent across custom events of a single component. Additionally, some extra care must go into merging the given ref and the ref managed by the component.

```tsx
import { dispatchCustomEvent } from 'react';


const MyComponent = React.forwardRef(({ onCustomEvent }, ref) => {
  // Omitting the implementation of the `useOptionalRef` hook because it is irrelevant
  // The purpose is to show that some care must go into merging the given ref
  // and the ref used in `dispatchCustomEvent`, as the `ref` provided may be null or undefined
  const targetRef = useOptionalRef(ref)

  useImperativeHandle(targetRef, () => {
    performSomeComponentAction: () => {}
    /* generate handle*/
  })

  React.useEffect(() => {
    subscribeToSomething(() => {
      dispatchCustomEvent(onCustomEvent, {
        type: 'custom-event'
        targetRef
      })
    })
  }, [])
})
```

##### Personal preference

My personal preference is "Option 2: new hook for defining imperative handle that emits custom events" for the following reasons.

1. It seems it would be relatively simple to implement
2. Provides a clear pattern for binding the target
3. Makes it easy to expose the ref to the caller

#### API 2: Component consumers use the event target

This demo is rather simple as it just shows what consumers of the component will be able to do once the target is exposed. Following the example implementations above, consumers should be able to write the following code:

```tsx
<MyComponent
  onCustomEvent={(evt) => {
    if (someCondition) {
      evt.target.performSomeComponentAction();
    }
  }}
/>
```

### Final Types

It was difficult expressing the design ideas without expressing the types in incremental parts. However, I will choose to provide the type definitions in a centralized location here for reference.

I've also tried to break it up into independent features

```ts
//// CORE FEATURES
interface CustomEvent<DetailType, TargetType> {
  readonly type: string;
  readonly detail: DetailType;
  readonly cancelable: boolean;
  readonly timeStamp: number;

  readonly isTrusted: false;
  readonly bubbles: false;
  readonly composed: false;
}

type CustomEventHandler<DetailType, TargetType> = (evt: CustomEvent<DetailType, TargetType>) => void;

interface CustomEventInit<DetailType, TargetType> {
  type: stirng;
  detail?: DetailType;
}

interface DispatchCustomEvent {
 <DetailType, TargetType>(handler: CustomEventHandler<DetailType, TargetType>, eventInit: CustomEventInit<DetailType, TargetType>): void
}


const dispatchCustomEvent: DispatchCustomEvent;

//// EVENT DEFAULT BEHAVIOR / CANCELABLE FEATURES
interface CustomEvent<DetailType, ValueType> {
  readonly cancelable: boolean;
  readonly preventDefault(): void;
  readonly isDefaultPrevented(): boolean;
  readonly defaultPrevented: boolean;
}

interface CustomEventInit<DetailType, ValueType> {
  defaultBehavior?(): void;
  cancelable?: boolean;
}


//// IMPERATIVE EVENT API FEATURES
interface CustomEvent<DetailType, TargetType> {
  target: TargetType;
}

interface CustomEventInit<DetailType, TargetType> {
  target?: TargetType;
}

// Assuming the closed API described above
type BoundCustomEventInit<DetailType> = Omit<CustomEventInit<DetailType, any>, 'target'>
type BoundDispatchCustomEvent<DetailType, TargetType> = (
  handler: CustomEventHandler<DetailType, TargetType>,
  eventInit: BoundCustomEventInit<DetailType, TargetType>
): void
type ImperativeHandleWithBoundTarget<DetailType, TargetType> = {
  dispatchCustomEvent: BoundDispatchCustomEvent<DetailType>
}
type UseEventTargetImperativeHandle<DetailType, TargetType> = (
    ref: React.Ref<TargetType>,
    handleFactory: () => TargetType,
    dependencies: any[]
  ) => ImperativeHandleWithBountTarget<DetailType, TargetType>;
```

### Alternatives

- User land solution. Because this doesn't require changes to the core of react, this can be easily handled by an open source library. However, it is less likely to achieve the success I would like to see with this as the chances of multiple open source libraries implementing slightly different patterns is still there. The problem is less in the complexity of the implementation, but in the lack of a standard, which leads to the inconsistencies I mentioned in the problem statement.

- [RFC: EventTarget](https://github.com/reactjs/rfcs/pull/246) has some overlaps with this RFC. However, I believe this proposal align with the react conventions for custom events and is simpler to implement because it does not rely on props requiring special treatment by react.

# Drawbacks

<!--
Why should we _not_ do this? Please consider:

- implementation cost, both in term of code size and complexity
- whether the proposed feature can be implemented in user space
- the impact on teaching people React
- integration of this feature with other existing and planned features
- cost of migrating existing React applications (is it a breaking change?)

There are tradeoffs to choosing any path. Attempt to identify them here.
-->

I may lean on the react team for a more thorough exploration of the drawbacks. These are some of the drawbacks I was able to identify.

1. The API for emitting custom events adds some complexity to something that is relatively simple for react users: Calling a callback function.
1. Since it is an opt-in feature and it may be hard for casual users of react to see the benefit of using this API, it may go unused in the majority of cases.
1. As I mentioned before, this can be implemented in the user space. However, `react` is positioned to have a broader and more positive impact than any open source library.
1. Library developers that choose to adopt these new APIs will have to consider potential breaking changes. Older unmaintained libraries will continue to expose inconsistent APIs.
1. This solution is attempting to solve an inconsistency problem, but since it is an opt-in solution, it will never completely get rid of inconsistency in library APIs.

# Adoption strategy

<!-- If we implement this proposal, how will existing React developers adopt it? Is
this a breaking change? Can we write a codemod? Should we coordinate with
other projects or libraries? -->

For react this would not be a breaking change.

For library developers choosing to implement a custom event API that aligns with react and this proposal, they will likely have to introduce a breaking change in their libraries, unless the implement some strategy for maintaining backwards compatibility (like emitting the legacy and new events)

# How we teach this

<!-- What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing React patterns?

Would the acceptance of this proposal mean the React documentation must be
re-organized or altered? Does it change how React is taught to new developers
at any level?

How should this feature be taught to existing React developers? -->

Thankfully much of this proposal builds on existing web standards which may make it easy to teach. However, recognizing that there will be differences between the web standards and the APIs expressed here, I'd imagine that we would call this "React Custom Events".

# Unresolved questions

<!-- Optional, but suggested for first drafts. What parts of the design are still
TBD? -->

I will step away from the problem for a few and revisit this if I encounter any doubts 😅
