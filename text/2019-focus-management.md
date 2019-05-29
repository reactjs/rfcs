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
      <FocusScope contain restoreFocus>
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

### Focus containment

Focus containment is important for modals and other popups. It prevents users from tabbing or focusing elements outside a particular region. For example, if a user tabs to the last element in a modal dialog, when they hit tab again the first element of the dialog should be focused instead of something outside. This is currently impossible to implement in a general way in React without manual DOM querying and imperative focusing.

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

In addition, portals make implementing focus containment in user space difficult since it is impossible to know whether a DOM element (e.g. dialog) “contains” another element (e.g. overlay) in the original React tree since portals may actually mount that element outside the dialog DOM tree altogether.

# Detailed design

Two public APIs in `react-dom` will be introduced in this RFC: `FocusScope` for declararing a scope of focusable elements, and `FocusManager` for programmatic movement of focus.

## Definitions

The following terms will be used throughout this RFC.

- A **focusable** element is any DOM element which has a `tabIndex` property, along with a set of default elements such as `input`, `button`, etc.
- A **tabbable** element is any DOM element which has a `tabIndex` value `>= 0`, along with a set of default elements such as `input`, `button`, etc.

## The focus tree

Each React root will have an internal focus tree managed by `react-dom`. At the root is an implicit `FocusScope` node, which can contain focusable elements and other focus scopes. The focusable elements form an ordered list within a focus scope, which is used when tabbing around with the keyboard. These elements are automatically added to the nearest focus scope by react-dom as they are mounted.

Focus scopes group focusable elements together. They remember the focused element within the scope, even when it is not currently focused. This allows focus to be restored to the correct child element when the scope regains focus, for example when tabbing back to a list of items, or closing a dialog.

When a `FocusScope` which contains the currently focused element unmounts, the element that previously had focus can optionally regain focus, if the `restoreFocus` prop is set. This means that the stored last focused item in that scope will be focused. For example, when clicking on a button to open a dialog, that button gets focused within its scope. When the dialog scope unmounts, focus is restored to the button.

Focus scopes can also be used to contain focus within a scope. If the `contain` prop is passed to the `FocusScope` component, then focus is contained within that scope. This prevents the user from tabbing out of that focus scope. If the user tabs to the last item in the scope, then the next tab will cycle to the first item in the scope.

If the `contain` prop is not passed to the `FocusScope`, or the scope is the implicit root `FocusScope`, then the user can tab in and out of the scope at will. Once focus is inside the scope, components can use the `FocusManager` API to programmatically move focus within the scope, for example in response to arrow keys or other interactions. This exact behavior is not provided by React, but APIs to perform these actions are available for component libraries to use. See below for details.

The `autoFocus` prop can also be passed to a `FocusScope`, which automatically focuses the first focusable element within that scope on mount.

The following is the complete interface for the `FocusScope` component.

```jsx
type FocusScopeProps = {
  contain?: boolean,
  restoreFocus?: boolean,
  autoFocus?: boolean
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

How the next, previous, first, and last elements are computed depends on the focus tree, and the currently focused element. In general, the list of candidates includes all elements within the currently focused scope, and for tab stops, all parent scopes until a scope with containment enabled or the root is reached. If no scope currently has focus, then the first or last element of the root scope receives focus according to the called method.

One obvious missing method in this interface is a method to focus a specific element. That case is already well covered by refs. There is no other good way to refer to a specific React element in a component in order to focus it.

## Implementation

This RFC can be implemented in `react-dom` by handling the Tab key using a global keyboard event handler. If the target of the event is within the `FocusScope`, the original event is canceled and the next element in the tab sequence computed from the React tree is focused programmatically. When the `contain` prop is enabled, upon reaching the last tabbable element in the scope, the focus wraps around to the first tabbable element in the scope. When the `contain` prop is not enabled, focus is allowed to continue to the next `FocusScope` in the tree.

When the Shift key is held when pressing the Tab key, focus moves in the opposite direction. When the `contain` prop is enabled, and focus reaches the first tabbable element in the scope, focus wraps to the last tabbable element in the scope. Since some browsers have additional focusing behavior that users expect, if any other modifier keys are held (e.g. Alt, Ctrl, etc.), then React should do nothing and leave the event for the browser to handle.

While handling the Tab key may not cover all possible cases across all browsers and platforms, it is the only reliable way to implement this behavior without relying on features like positive `tabIndex` (which can cause other accessibility problems) or additional "sentinel" DOM nodes (which can affect e.g. CSS selectors). If a platform implements focusing behavior that does not fire Tab key events, then the behavior would be exactly as it is today - following DOM order rather than React tree order. Additional support for these platforms could be added to React in the future as well if they are deemed important enough to support.

When the `restoreFocus` prop is enabled, the last focused element is stored by React when the `FocusScope` mounts. When the `FocusScope` unmounts, focus is restored to that element. If the element is no longer in the DOM, then focus is restored to the body (the default browser behavior).

The `FocusManager` API could be implemented using the same internals as `FocusScope`. Since it is a singleton, it first finds the currently focused `FocusScope`, and applies all actions to that scope. For example, the `focusNextTabStop` function would follow the same steps above as if a user pressed the Tab key.

## Examples

The following are examples that solve the initially discussed challenges, using the proposed API.

### Refs everywhere

While this challenge is not completely eliminated since refs may still be necessary to focus specific elements, there should be fewer refs and manual DOM querying than before since many things can be accomplished with the `FocusManager` and `FocusScope` APIs instead.

### Global focus state

This issue is solved by the `FocusScope` API, which remembers focused elements within only that scope, rather than components needing to implement this manually.

### Arrow key navigation

The following example shows how a keyboard accessible list component might be implemented. It uses a `FocusScope` to group the elements together. When the arrow keys are pressed, the `FocusManager` API is used to move focus to the previous or next focusable item in the list. The roving tab index pattern is implemented to ensure that only one item is tabbable at a time. Initially, all `li` elements have `tabIndex="0"` to make them tabbable. On focus, the last focused item is stored in local component state. This causes the `tabIndex` of the last focused element to become "0" and all others to become "-1". If the tab key is pressed, the list item loses focus. When tabbing back to the list, focus is restored to the item with "tabIndex="0" by the browser.

This example is similar to how an accessible list would be implemented today, but avoids refs and manual querying of the DOM to move focus on keyboard events.

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

  let [lastFocusedItem, setLastFocusedItem] = useState(null);
  let onFocus = (item) => {
    setLastFocusedItem(item);
  };

  return (
    <FocusScope>
      <ul role="listbox" onKeyDown={onKeyDown}>
        {items.map((item, index) => (
          <li
            role="option"
            key={index}
            tabIndex={!lastFocusedItem || lastFocusedItem === item ? 0 : -1}
            onFocus={() => onFocus(item)}>
            {item}
          </li>
        ))}
      </ul>
    </FocusScope>
  );
}
```

### Focus containment

The following implements a generic dialog component which contains focus within it while open, and automatically restores focus to the previously focused element when closed. This example would require hundreds of lines of user space code to implement properly outside of React core.

```jsx
function Dialog({ children }) {
  return (
    <Portal>
      <FocusScope contain restoreFocus>
        <div className="dialog">
          {children}
        </div>
      </FocusScope>
    </Portal>
  );
}
```

### Restoring focus

Building on the previous example, this example shows how one might open a dialog. Since the `restoreFocus` prop is passed, when the `FocusScope` within the dialog unmounts as the dialog closes, focus is automatically restored to the last focused item in the parent scope, in this case the button.

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

The original portals example would have the correct behavior without code changes. React would handle the Tab key such that it appears in the tab order based on the React tree rather than the DOM tree.

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
