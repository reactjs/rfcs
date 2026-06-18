# RFC: `useSuppressKeyboard`

- Start Date: 2026-06-18
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

---

# Summary

`useSuppressKeyboard` is a proposed React hook that provides a first-class,
declarative API for preventing the mobile virtual (on-screen) keyboard from
appearing when a focusable element receives focus. It abstracts away a
collection of fragile, platform-inconsistent DOM workarounds behind a stable,
well-specified interface that integrates naturally with React's rendering and
ref models.

---

# Basic Example

```jsx
import { useSuppressKeyboard } from 'react';

function DatePickerTrigger() {
  const ref = useSuppressKeyboard();

  return (
    <input
      ref={ref}
      type="text"
      readOnly
      placeholder="Select a date"
      onClick={openDatePicker}
    />
  );
}
```

A boolean `enabled` option allows the suppression to be toggled at runtime:

```jsx
function ConditionalInput({ isPickerMode }) {
  const ref = useSuppressKeyboard({ enabled: isPickerMode });

  return <input ref={ref} type="text" />;
}
```

The hook returns a `RefCallback` / `RefObject`-compatible ref that can be
passed directly to any host element. No wrapper components, no render props,
no imperative DOM mutations scattered across `useEffect` calls.

---

# Motivation

## The problem

Mobile browsers display a virtual keyboard whenever a focusable element—most
commonly `<input>`, `<textarea>`, or a `contenteditable` node—receives focus.
In the vast majority of cases this is exactly the right behaviour. There is,
however, a meaningful class of UI patterns where showing the keyboard is
actively harmful to the user experience:

| Pattern | Why the keyboard should stay hidden |
|---|---|
| Custom date / time pickers | The value is selected from a visual widget; raw text entry is neither desired nor valid. |
| Rich-text / code editors backed by a canvas or custom renderer | The editor owns key input but renders to a surface that is not a native text field. |
| Barcode / QR-code scanner fields | Input arrives from a hardware peripheral; the soft keyboard obscures the camera viewfinder. |
| Game or interactive-canvas elements that capture keyboard events | The element needs keyboard focus for hotkeys without triggering the layout reflow caused by the keyboard. |
| Read-only searchable lists | A `<div tabIndex={0}>` needs arrow-key navigation without a keyboard opening. |

In every case the developer needs focus (for accessibility and keyboard events)
but not the virtual keyboard.

## Current workarounds are fragile and non-composable

Without a platform primitive, React developers reach for a combination of
techniques, each with significant drawbacks:

### 1. `readOnly` + manual value management

Setting `readOnly` on an `<input>` suppresses the keyboard on most platforms.
However it also disables form validation, changes `onChange` semantics, and can
confuse screen readers. Developers end up toggling `readOnly` on and off in
`onFocus`/`onBlur` handlers, racing against browser focus events and React's
synthetic event system.

### 2. `inputMode="none"`

`inputMode="none"` is the closest existing HTML attribute to what this RFC
proposes. It suppresses the virtual keyboard on Android Chrome and is partially
respected elsewhere. However:

- It is **not** universally supported: Safari on iOS silently ignores it for
  `<input type="text">` elements (confirmed as of iOS 17).
- It has **no effect** on non-`<input>` focusable elements such as
  `contenteditable` divs or elements with `tabIndex`.
- It does not suppress the keyboard accessory bar that iOS renders above the
  keyboard even when `inputMode="none"` is partially honoured.
- It gives the developer no programmatic way to change behaviour at runtime
  based on component state.

### 3. `blur()`-on-focus hacks

Some libraries call `element.blur()` immediately after `element.focus()`,
betting that the brief focus/blur cycle prevents the keyboard from opening
without losing accessibility focus tracking. This is a race condition. It
breaks screen reader virtual cursors, invalidates `:focus-visible` styles, and
fails whenever the browser schedules its keyboard-show animation before the
blur fires.

### 4. `pointer-events: none` overlays

Placing an invisible overlay over the input to intercept taps, then manually
routing events: highly fragile, breaks native form autofill, and adds
meaningful accessibility surface area.

### 5. Scroll-position preservation hacks

When the keyboard opens it causes a viewport resize. Teams sometimes suppress
this indirectly by locking `document.body` scroll position. This is a symptom
treatment, not a fix.

## The cost of the status quo

Beyond correctness, the scattered workaround ecosystem has real maintenance
costs:

- Every date-picker, rich-text editor, and custom select library re-implements
  the same patterns independently and inconsistently.
- The patterns break silently across OS updates (Safari 17 changed keyboard
  suppression behaviour for `inputMode`).
- None of the patterns integrate with React's concurrent rendering model. A
  `readOnly` toggle inside a `useEffect` may run after a paint, causing a
  visible flash of the keyboard.
- Server-side rendering teams must guard every workaround with `typeof window`
  checks.

A standardised hook addresses all of these concerns in one place.

---

# Detailed Design

## Hook signature

```ts
type UseSuppressKeyboardOptions = {
  /**
   * When false the hook is a no-op and keyboard suppression is not applied.
   * Defaults to true.
   */
  enabled?: boolean;
};

function useSuppressKeyboard<T extends HTMLElement = HTMLElement>(
  options?: UseSuppressKeyboardOptions,
): React.RefCallback<T>;
```

The hook returns a **ref callback** (not a `RefObject`). This is intentional:
ref callbacks fire synchronously in the commit phase, before the browser has
had a chance to schedule a keyboard-open animation, giving React the earliest
possible insertion point.

## Implementation strategy

The hook's internal implementation uses a layered strategy, applying the
most-capable mechanism available in the current browsing environment and
falling back gracefully.

### Layer 1: `VirtualKeyboard.hide()` (where available)

The [VirtualKeyboard API](https://developer.mozilla.org/en-US/docs/Web/API/VirtualKeyboard_API)
(`navigator.virtualKeyboard`) is a modern web platform primitive that gives
precise programmatic control over the virtual keyboard. Where supported
(Chromium 94+), the hook will:

1. Set `navigator.virtualKeyboard.overlaysContent = true` on mount (once, per
   document).
2. Attach a `focus` listener to the element that calls
   `navigator.virtualKeyboard.hide()`.
3. Restore `overlaysContent` to its previous value on unmount.

This is the only approach that is both synchronous and lossless with respect to
focus state.

### Layer 2: `inputMode="none"` (HTMLInputElement / HTMLTextAreaElement)

For `<input>` and `<textarea>` elements on platforms that do not support the
VirtualKeyboard API, the hook will:

1. Store the element's current `inputMode` value.
2. Set `inputMode = 'none'` synchronously in the ref callback.
3. Restore the original value if `enabled` changes to `false` or on unmount.

This correctly handles the case where the consumer has also set an `inputMode`
prop; the hook always yields to an explicitly-set prop when suppression is
disabled.

### Layer 3: `contenteditable` and generic focusable elements

For elements that are neither `<input>` nor `<textarea>` (e.g., elements with
`tabIndex`, `contenteditable` divs), the hook will:

1. Temporarily set `contenteditable="false"` if the element is currently
   `contenteditable="true"`, then restore it after a microtask (this suppresses
   the keyboard on iOS Safari).
2. Attach an early-capture `touchstart` listener that calls
   `event.preventDefault()` when `enabled` is `true`, which prevents iOS
   Safari from raising the keyboard for non-input elements.

The `touchstart` suppression is limited to the hook's managed element and does
not affect event propagation elsewhere in the tree.

### Layer 4: No-op / graceful degradation

On platforms where none of the above techniques are applicable (desktop
browsers, environments with no virtual keyboard concept), the hook is a
transparent no-op. It still attaches the ref so that it can be used
unconditionally in universal components.

## Ref composition

The hook returns a `RefCallback`. To compose it with an existing `ref` prop,
developers use `React.composeRefs` (proposed alongside this RFC, or available
today via third-party utilities):

```jsx
const suppressRef = useSuppressKeyboard();
const myRef = useRef(null);

<input ref={composeRefs(suppressRef, myRef)} />;
```

Alternatively, the hook accepts an optional `ref` forwarding option as a
second argument:

```jsx
const ref = useSuppressKeyboard({ enabled: true }, forwardedRef);
```

## Interaction with `enabled` changes

When `enabled` changes from `true` to `false` between renders, the hook must
remove all mutations applied to the element. This is handled entirely within
the ref callback and a corresponding cleanup function—no `useEffect` is
needed, preserving compatibility with concurrent rendering and transitions.

## SSR behaviour

The hook renders as a no-op on the server. The returned ref callback is a
stable identity that performs no DOM mutations when attached. Hydration is
unaffected.

## Accessibility contract

The hook explicitly does **not** remove focus from the element. The element
remains in the document's focus order, reachable by keyboard navigation, and
correctly described by screen reader accessibility APIs. The _only_ side effect
is the suppression of the virtual keyboard's visual appearance. This is
analogous to `inputMode="none"` from an accessibility-model perspective.

Developers are responsible for ensuring that elements which suppress the
keyboard still provide an equivalent input pathway for users who rely on
alternative input methods. This requirement is documented in the hook's API
reference.

## Full example: custom date picker

```jsx
import { useRef, useState } from 'react';
import { useSuppressKeyboard } from 'react';

export function DateField({ value, onChange }) {
  const [pickerOpen, setPickerOpen] = useState(false);
  const inputRef = useSuppressKeyboard();

  return (
    <>
      <input
        ref={inputRef}
        type="text"
        readOnly
        value={value ?? ''}
        onFocus={() => setPickerOpen(true)}
        aria-haspopup="dialog"
        aria-expanded={pickerOpen}
      />
      {pickerOpen && (
        <CalendarDialog
          value={value}
          onChange={(d) => { onChange(d); setPickerOpen(false); }}
          onClose={() => setPickerOpen(false)}
        />
      )}
    </>
  );
}
```

---

# Drawbacks

This section is intentionally thorough. The author believes the drawbacks are
real and that reviewers should weigh them carefully before accepting this RFC.

## 1. `inputMode="none"` already exists

The most honest counter-argument to this RFC is that the web platform has
already addressed the keyboard-suppression use case via `inputMode="none"`.
For the majority of `<input>` and `<textarea>` elements on Android Chrome—the
most common combination—it works correctly today. React could simply improve
its documentation and provide a helper that sets `inputMode="none"` rather than
introduce a hook with a layered implementation.

The counter-counter-argument is that `inputMode="none"` does not work on iOS
Safari and does not generalise to non-input elements. But reasonable people can
disagree about whether that gap warrants new React core API.

## 2. Keyboard suppression is a platform responsibility

One could argue that the inconsistency in `inputMode="none"` support is a
browser bug, and that React should not paper over browser bugs with
abstractions—it should file browser issues and wait for the platform to
converge. The VirtualKeyboard API is already shipping in Chromium; there is a
reasonable expectation that other engines will follow.

Adding a React abstraction now could calcify workarounds that should be
temporary and could slow browser vendors' incentive to fix the underlying
platform inconsistency.

## 3. Implementation complexity vs. user-space feasibility

The layered implementation described above is non-trivial. It branches on
element type, browser capabilities, and platform. More critically, this logic
can be implemented entirely in user space today—and libraries like
`react-datepicker`, `react-select`, and `@floating-ui/react` already ship
versions of it. Adding this to React core means React owns the maintenance
burden for browser-specific behaviour that changes frequently.

## 4. Risk of misuse

A `useSuppressKeyboard` hook with a friendly name and low friction is easy to
reach for in cases where the keyboard _should_ appear. Developers building
accessible forms might suppress the keyboard on inputs they deem "read-only"
without providing an alternative input method, creating real barriers for users
who depend on virtual keyboards as their primary input mechanism (users with
motor disabilities, for example).

The HTML attribute `inputMode="none"` carries with it a certain intentionality
and specificity—you have to know what you're doing to type `inputMode="none"`.
A hook with a memorable name lowers that bar.

## 5. Scope creep concerns

The VirtualKeyboard API is rich: it exposes geometry information, resize
events, and overlay modes that go well beyond suppression. A `useSuppressKeyboard`
hook scoped only to hiding the keyboard may create pressure for follow-on
hooks (`useVirtualKeyboardGeometry`, `useVirtualKeyboardVisible`, etc.) that
should perhaps be addressed as a cohesive VirtualKeyboard API rather than
piecemeal hooks.

## 6. Testing and CI overhead

Platform-specific virtual-keyboard behaviour cannot be exercised in jsdom. Any
test suite that wants to verify keyboard suppression must run on real devices
or browser automation with a simulated keyboard (which most CI pipelines do not
have). React's existing test infrastructure does not include device-farm
integration, meaning this feature would ship with meaningful gaps in automated
test coverage.

---

# Alternatives

## Do nothing / document `inputMode="none"` better

The simplest alternative: improve React's documentation to explain
`inputMode="none"`, its limitations on iOS Safari, and the workarounds. Provide
a copy-pasteable user-space hook in the documentation rather than including one
in React core.

**Impact of not doing this:** Status quo. Teams continue to implement their own
workarounds, inconsistently. Library authors continue to ship divergent
implementations. iOS Safari continues to be a pain point.

## Ship a standalone `use-suppress-keyboard` npm package

Publish the layered implementation described in this RFC as a standalone
`react-suppress-keyboard` package under the `@react-ecosystem` scope (or as a
community package). This keeps React core lean, allows faster iteration, and
moves the browser-compatibility burden outside of the React release cycle.

This is the author's preferred alternative if the RFC is not accepted.

## Advocate for the VirtualKeyboard API in all browsers

File or upvote browser bugs for VirtualKeyboard API support in Safari and
Firefox. If all major browsers supported `navigator.virtualKeyboard`, the
correct solution for React would be a thin one-liner wrapper rather than a
layered abstraction.

## Extend `useRef` with an `inputMode` option

A narrower alternative that sidesteps the multi-platform complexity: provide an
`options` bag to a new `useHostRef` primitive that allows declarative attribute
control, of which `inputMode` is one possible entry. This would be more
general-purpose but would not solve the iOS Safari gap.

---

# Adoption Strategy

This is a new, additive hook. It introduces no breaking changes.

- Existing code that uses `readOnly` or `inputMode="none"` workarounds
  continues to work unchanged.
- Teams can migrate incrementally on a component-by-component basis.
- No codemod is required. A lint rule could optionally detect common
  `readOnly`-toggle patterns and suggest `useSuppressKeyboard` as a
  replacement, but this would be advisory only.
- Library authors (`react-datepicker`, `react-select`, etc.) are the primary
  adoption target. Coordinating with major library maintainers before shipping
  would increase the probability of consistent ecosystem-wide adoption.

---

# How We Teach This

## Naming

`useSuppressKeyboard` follows React's established `use*` hook naming convention
and uses plain English that communicates intent without requiring background
knowledge of the VirtualKeyboard API. Alternative names considered:

- `useNoVirtualKeyboard` — slightly awkward phrasing.
- `useInputMode` — too narrow (implies it only works on `<input>`).
- `useVirtualKeyboard` — too broad (implies full VirtualKeyboard API access).

## Documentation placement

The hook should be documented in the **Hooks Reference** section alongside
other DOM-integration hooks (`useRef`, `useImperativeHandle`). A dedicated
guide page under **Integrating with the DOM** should explain the virtual
keyboard problem space, the limitations of `inputMode="none"`, and when to
reach for this hook vs. the raw HTML attribute.

## Teaching framing

The hook should be introduced as a **progressive enhancement primitive**:
developers who already understand `inputMode` can adopt it immediately; those
new to the problem space can follow the guide page's explanation of why virtual
keyboard management matters for custom form controls.

It should be taught alongside the broader concept of "focus management in
custom form controls" rather than in isolation, to avoid encouraging its
misuse.

## Impact on existing curriculum

The React documentation section on forms currently does not cover virtual
keyboard management at all. Accepting this RFC would prompt an addition to that
section but would not require reorganisation of existing content.

---

# Unresolved Questions

1. **Should the hook accept an optional existing ref?** The current design
   returns a new `RefCallback`. If the consumer already has a `useRef`, they
   must compose the two refs manually. Should the hook instead accept an
   optional `existingRef` argument and merge internally?

2. **Should `overlaysContent` be managed globally or per-element?**
   `navigator.virtualKeyboard.overlaysContent` is a document-level setting. If
   multiple components on the same page use `useSuppressKeyboard`, their
   toggling of this property could conflict. The implementation would need a
   reference-counted global store, similar to how `document.body.style.overflow`
   locking is conventionally managed. The correct approach has not been fully
   specified.

3. **What is the correct behaviour when `enabled` toggles while the element
   has focus?** If the keyboard is currently suppressed and `enabled` flips to
   `false`, should the keyboard immediately appear? This would be surprising in
   most cases but may be correct in others (e.g., switching from picker mode to
   free-text mode mid-interaction).

4. **Should this hook generalise to `<dialog>` elements and other focus
   recipients?** The current scope is limited to form-like elements, but the
   same problem exists for modal dialogs that capture keyboard events without
   wanting a virtual keyboard.

5. **Relationship to the Pointer Events specification and `touchstart`
   suppression:** Using `event.preventDefault()` on `touchstart` has
   side-effects beyond keyboard suppression (it also prevents scroll). The
   fallback Layer 3 implementation must be audited to confirm it does not
   inadvertently break scroll behaviour on elements inside scrollable
   containers.

6. **Platform detection vs. feature detection:** The layered implementation
   relies on feature detection (`'virtualKeyboard' in navigator`), not
   user-agent sniffing. However, some of the fallback layers (particularly the
   iOS-specific `contenteditable` trick) require knowledge of the platform.
   The implementation should be audited to ensure it uses feature detection
   exclusively.
