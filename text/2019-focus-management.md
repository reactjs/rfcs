- Start Date: 2019-02-27
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

This RFC proposes adding a builtin `FocusScope` component and `FocusManager` API to `react-dom`, along with internal logic to properly manage focus in React applications. Along with focus scopes declared in the application, it will take into consideration portals and other React concepts which may cause the React tree and the DOM tree to differ in relation to focus management.

# Basic example

This example shows how a dialog component might be implemented using the proposed `FocusScope` API.

```jsx
import { FocusScope } from 'react-dom';

function Dialog({ children }) {
  return (
    <Portal>
      <FocusScope lock>
        <div className="dialog">
          {children}
        </div>
      </FocusScope>
    </Portal>
  );
}

function App() {
  let [showDialog, setShowDialog] = useState(false);

  return (
    <div>
      <button onClick={() => setShowDialog(true)}>Open Dialog</button>
      {showDialog &&
        <Dialog>
          <input placeholder="input 1" />
          <input placeholder="input 2" />
          <button onClick={() => setShowDialog(false)}>Close Dialog</button>
        </Dialog>
      }
    </div>
  );
}
```

# Motivation

Focus management is the programmatic movement of keyboard focus in an application in response to user input, such as mouse, touch, or keyboard interactions. Implementing keyboard support for components and applications is imperative for accessibility, and not enough web applications implement this properly today. See the [ARIA Practices](https://www.w3.org/TR/wai-aria-practices-1.1/#keyboard) document for more information about focus management and keyboard interfaces.

In React, implementing many common components is very easy, but implementing those components with proper support for accessibility and focus management is still quite difficult. Applications that do implement support typically use a component library which has taken the time to get this right. However, implementing such a library correctly is very challenging and time consuming, and React could enable library authors to handle focus management in an easier way.

## Challenges

When building a library of accessible components implementing ARIA patterns, it is necessary to implement proper focus management. Today, this is quite difficult to do correctly due to the following challenges.

### Refs everywhere

Refs are meant as an escape hatch from the declarative React model to make things happen imperatively, but when implementing accessible components it is quite common to need one or more in each component. One of the main reasons to have refs is for focus management. Particular elements need to be focused in response to events like keyboard navigation with arrow keys, etc. In addition, sometimes specific children of refs need to be found with `querySelector` and other DOM methods in order to focus the correct elements, e.g. elements inside child components. This can feel unclean, and un-reacty.

### Global focus state

The imperative DOM focus management API is global: only one element can be focused at a time. Calling `node.focus()` immediately causes all other nodes to lose focus, and the new node to gain focus. Therefore, components basically fight one another for focus. In reusable components, this is a challenge since they don't know what other components are on a page. There is no way for a component to "remember" what was focused before, so that focus can be restored to it later without some manual work in every component. This will be described more below in some of the other sections.

### Arrow key navigation

Keyboard users typically navigate between components with the tab key, which changes the focused element. However, composite components such as lists, grids, trees, etc. should only appear in the tab sequence once. It would be annoying if you had to tab through every item in a long list just to get past the entire list to the next element afterward. Therefore, you tab to the list, and if you want to navigate within the list you use the arrow keys, and otherwise you can tab to the next element after the entire list.

The [roving tab index](https://www.w3.org/TR/wai-aria-practices-1.1/#kbd_general_within) pattern is one way to solve this problem, but basically it involves each component being able to "remember" what was focused last, so that if a user tabs away from a list and then back, the list item they were on before is re-focused rather than the first item or the list itself.

### Focus isolation

Focus isolation is important for modals and other popups. It prevents users from tabbing or focusing elements outside a particular region. For example, if a user tabs to the last element in a modal dialog, when they hit tab again the first element of the dialog should be focused instead of something outside. This is currently impossible to implement in a general way in React without manual DOM querying and imperative focusing.

### Restoring focus

When you close a dialog or popover, focus should be restored to whatever had focus before the modal opened. This requires us to remember what was focused last, and when the component unmounts, focus that element again.

### Portals

React portals mean that the order of elements in the DOM might not match the order of elements in the React tree. This can cause unintended issues with focus because the browser does not know the correct order to tab through elements. This can be seen with a simple example:

```jsx
function App() {
  return (
    <div>
      <input placeholder="input 1" />
      <Portal>
        <input placeholder="input 2" />
      </Portal>
      <input placeholder="input 3" />
    </div>
  );
}
```

The tab order in this example will be input 1, input 3, and then input 2, assuming the portal is placed after the app in the DOM. Users may expect the order to be input 1, input 2, input 3, as declared in the React tree. The current behavior is non-deterministic — it depends on the specific implementation of the Portal component (i.e. where it is placed in the DOM), which could change over time and cause the tab order to differ from what was intended. It also depends on the order portals are appended to the DOM rather than the order they are declared. These issues would be solved if the tab order in portals were based on the order in the React tree rather than the DOM tree. This would also match other existing portal behavior such as event bubbling where portals behave as if they are inline.

In addition, portals make implementing focus isolation in user space difficult since it is impossible to know whether a DOM element (e.g. dialog) “contains” another element (e.g. overlay) in the original React tree since portals may actually mount that element outside the dialog DOM tree altogether.

# Detailed design

Two public APIs in `react-dom` will be introduced in this RFC: `FocusScope` for declararing a scope of focusable elements, and `FocusManager` for programmatic movement of focus.

## Definitions

The following terms will be used throughout this RFC.

- A **focusable** element is any DOM element which has a `tabIndex` property, along with a set of default elements such as `input`, `button`, etc.
- A **tabbable** element is any DOM element which has a `tabIndex` value other than `-1`, along with a set of default elements such as `input`, `button`, etc.

## The focus tree

Each React root will have an internal focus tree managed by `react-dom`. At the root is an implicit `FocusScope` node, which can contain focusable elements and other focus scopes. The focusable elements form an ordered list within a focus scope, which is used when tabbing around with the keyboard. These elements are automatically added to the nearest focus scope by react-dom as they are mounted.

Focus scopes group focusable elements together. They remember the focused element within the scope, even when it is not currently focused. This allows focus to be restored to the correct child element when the scope regains focus, for example when tabbing back to a list of items, or closing a dialog.

When a `FocusScope` which contains the currently focused element unmounts, the parent `FocusScope` in the focus tree regains focus. This means that the stored last focused item in that scope will be focused. For example, when clicking on a button to open a dialog, that button gets focused within its scope. When the dialog scope unmounts, focus is restored to the button.

Focus scopes can also be used to lock focus within that scope. If the `lock` prop is passed to the `FocusScope` component, then focus is locked within that scope. This prevents the user from tabbing out of that focus scope, and makes elements outside the scope unfocusable. If the user tabs to the last item in the scope, then the next tab will cycle to the first item in the scope.

If the `lock` prop is not passed to the `FocusScope`, or the scope is the implicit root `FocusScope`, then the user can tab in and out of the scope at will. Once focus is inside the scope, components can use the `FocusManager` API to programmatically move focus within the scope, for example in response to arrow keys or other interactions. This exact behavior is not provided by React, but APIs to perform these actions are available for component libraries to use. See below for details.

Unlike the DOM, `tabIndex` values on elements are scoped within a `FocusScope`. If a specific tab order is required, `tabIndex` can be set to a positive value. This works just like the default browser behavior, except the values are scoped within the nearest `FocusScope`. This allows the tab order to be controlled within a specific component without overriding the order for the entire page.

The `FocusScope` component itself also supports a `tabIndex` property, which when set to `"0"` or a positive value makes the `FocusScope` tabbable. When the `FocusScope` is tabbed to, it automatically focuses the previously focused child, or if it has never been focused before, the first focusable child. This is useful for components like lists, where only the last focused list item should be tabbable, but the rest should be focusable. See the “arrow key navigation” section under motivation above, and the example below for more details.

The following is the complete interface for the `FocusScope` component.

```jsx
type FocusScopeProps = {
  lock?: boolean,
  tabIndex?: number
};

interface FocusScope extends React.Component<FocusScopeProps> {}
```

## FocusManager

In addition to defining the focus scopes in an application, components need an API to programmatically move focus around. The singleton `FocusManager` API exported by `react-dom` provides this interface. 

```jsx
interface FocusManager {
  /* Focus the next focusable element in the current scope. */
  focusNext(): void;

  /* Focus the previous focusable element in the current scope. */
  focusPrevious(): void;

  /* Focus the first focusable element in the current scope. */
  focusFirst(): void;

  /* Focus the last focusable element in the current scope. */
  focusLast(): void;

  /* Focus the next tabbable element after the currently focused one. */
  focusNextTabStop(): void;

  /* Focus the previous tabbable element after the currently focused one. */
  focusPreviousTabStop(): void;

  /* Focus the first tabbable element. */
  focusFirstTabStop(): void;

  /* Focus the last tabbable element. */
  focusLastTabStop(): void;
}
```

How the next, previous, first, and last elements are computed depends on the focus tree, and the currently focused element. In general, the list of candidates includes all elements within the currently focused scope, and for tab stops, all parent scopes until a locked scope or the root is reached. If no scope currently has focus, then the first or last element of the root scope receives focus according to the called method.

One obvious missing method in this interface is a method to focus a specific element. That case is already well covered by refs. There is no other good way to refer to a specific React element in a component in order to focus it.

## Implementation

There are several ways that tab handling could be implemented internally in `react-dom`. The obvious way is for `react-dom` to take over handling of the tab key for the page in order to use the order defined in the React tree rather than the DOM tree. However, recreating browser behavior will cause problems. The tab key is not the only way that focus can be moved. Assistive technology like screen readers also look at DOM order when determining how to move focus. Without some way of exposing the tab order computed by React to the actual DOM, screen readers will rely on the same incorrect tab order as today.

Another way for `react-dom` to implement this is by computing the [tabIndex](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/tabindex) property for each tabbable element. `tabIndex` allows for positive values along with the more common `"0"`  and `"-1"`, which control the exact tab order that will be followed when using the tab key and screen readers. The example given in the motivation section above could be “fixed” by adding specific `tabIndex` values to each of the `input` elements, so that the tab order is input 1, input 2, input 3.

```jsx
function App() {
  return (
    <div>
      <input placeholder="input 1" tabIndex="1" />
      <Portal>
        <input placeholder="input 2" tabIndex="2" />
      </Portal>
      <input placeholder="input 3" tabIndex="3" />
    </div>
  );
}
```

This is very bug prone to implement manually in a reusable component since the sequence is global to the page. However, since `react-dom` has a view of the whole world, it could compute the desired tab order for elements based on the internal focus tree, and generate `tabIndex` values for each focusable element automatically. Elements outside of a locked `FocusScope` would get `tabIndex="-1"` so that they could not be tabbed to. This would solve the tab ordering and locking issues while not reimplementing any browser behavior and still working correctly with screen readers.

This implementation could be controversial, as positive `tabIndex` values are generally strongly discouraged. Some tooling [has rules](https://dequeuniversity.com/rules/axe/1.1/tabindex) to discourage developers from doing this, which would fail for React apps that utilize this. It could also break the tab order for pages where React is embedded and does not control the whole page. This could be solved by only assigning positive `tabIndex` values while focus is inside the React root, and also only when the order in the React tree would differ from the default DOM order due to portals. Alternative implementation strategies could also be considered, but it may be challenging to implement correctly without duplicating lots of default browser behavior and while continuing to work properly for screen readers.

## Examples

The following are examples that solve the initially discussed challenges, using the proposed API.

### Refs everywhere

While this challenge is not completely eliminated since refs may still be necessary to focus specific elements, there should be fewer refs than before since many things can be accomplished with the `FocusManager` and `FocusScope` APIs instead.

### Global focus state

This issue is solved by the `FocusScope` API, which remembers focused elements within only that scope, rather than components needing to implement this manually.

### Arrow key navigation

The following example shows how a keyboard accessible list component might be implemented. It uses a `FocusScope` to group the elements together. The `FocusScope` has `tabIndex="0"` so it is tabbable. The `li` elements have `tabIndex="-1"` so they are focusable but not tabbable. When the `FocusScope` is tabbed to, it automatically focuses the first list item. When the arrow keys are pressed, the `FocusManager` API is used to move focus to the previous or next focusable item in the list. If the tab key is pressed, the list loses focus. When tabbing back to the list, the `FocusScope` automatically restores focus to the list item that had focus last.

```jsx
import {FocusScope, FocusManager} from 'react-dom';

function List({ items }) {
  let onKeyDown = (e) => {
    switch (e.key) {
      case 'ArrowDown':
        FocusManager.focusNext();
        break;
      case 'ArrowUp':
        FocusManager.focusPrevious();
        break;
    }
  };

  return (
    <FocusScope tabIndex="0">
      <ul onKeyDown={onKeyDown}>
        {items.map((item, index) => (
          <li key={index} tabIndex="-1">{item}</li>
        ))}
      </ul>
    </FocusScope>
  );
}
```

### Focus isolation

The following implements a generic dialog component which locks focus within it while open, and automatically restores focus to the previously focused element when closed. This example would require hundreds of lines of user space code to implement properly outside of React core.

```jsx
function Dialog({ children }) {
  return (
    <Portal>
      <FocusScope lock>
        <div className="dialog">
          {children}
        </div>
      </FocusScope>
    </Portal>
  );
}
```

### Restoring focus

Building on the previous example, this example shows how one might open a dialog. When the `FocusScope` within the dialog unmounts as the dialog closes, focus is automatically restored to the last focused item in the parent scope, in this case the button.

```jsx
function App() {
  let [showDialog, setShowDialog] = useState(false);

  return (
    <div>
      <button onClick={() => setShowDialog(true)}>Open Dialog</button>
      {showDialog &&
        <Dialog>
          <input placeholder="input 1" />
          <input placeholder="input 2" />
          <button onClick={() => setShowDialog(false)}>Close Dialog</button>
        </Dialog>
      }
    </div>
  );
}
```

### Portals

The original portals example would have the correct behavior without code changes. The input within the portal would have a `tabIndex` set on it while focus is inside the React root such that it appears in the tab order based on the React tree rather than the DOM tree.

# Drawbacks

- It may be complex to implement. I’m not familiar enough with `react-dom`'s internals to say.
- The behavior differs from the default DOM behavior, and current React behavior, which could potentially be unexpected for users or break existing apps that rely on the current behavior.

# Alternatives

There have been many libraries developed in user space to implement focus management. However, none of them can fully implement the desired behavior without access to the React tree.

An alternative which would allow this proposal to be fully implemented in user space would be an API for accessing the full React tree from components at runtime. However, even if that existed, the proposal could be more efficiently implemented within React itself.

# Adoption strategy

`FocusScope` and `FocusManager` are opt-in APIs which users can chose to use or not. No code changes to existing React applications are required when upgrading to a version that supports these APIs.

The behavior of tabbing through applications that utilize portals would change, however. If applications currently rely on the existing behavior, it could be considered a “breaking” change.

# How we teach this

There should be a page in the React documentation to explain the `FocusScope` and `FocusManager` APIs. It should include examples of building common components like dialogs and lists in a keyboard accessible way.
