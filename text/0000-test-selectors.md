- Start Date: 2022-06-07
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

This proposal introduces a set of APIs to help with automated testing of React applications.

# Basic example

Given a website with the following content:
```html
<body>
  <!-- ... -->
  <main>
    <h1>React RFCs</h1>
    <article>
      <h2>React test selectors</h2>
      <p>
        This proposal introduces a set of APIs to help with automated testing of React applications.
        <a href="...">Learn more</a> <!-- We want to select this link -->
      </p>
    </article>
    <article>
      <h2>Server errors in React 18</h2>
      <p>
        This RFC describes a new recovery mechanism for server errors in React 18.
        It lets you recover from errors thrown on the server by adding Suspense boundaries.
        <a href="...">Learn more</a>
      </p>
    </article>
  </main>
  <!-- ... -->
</body>
```

We could use the test selector API to find the "learn more" link for this RFC:
```js
import {
  createHasPseudoClassSelector,
  createRoleSelector,
  createTextSelector,
  findAllNodes,
} from "react-dom/testing";

const matches = findAllNodes(document.body, [
  // Narrow down to articles within our page
  createRoleSelector("article"),

  // Focus on the article that is about test selectors
  createHasPseudoClassSelector([
    createRoleSelector("heading"),
    createTextSelector("React test selectors"),
  ]),

  // Select the "learn more" link
  createRoleSelector("link"),
  createTextSelector("Learn more"),
]);

// Verify that we've found one (and only one) matching link
expect(matches).toHaveLength(1);

// ...
```

# Motivation

There are testing framework that enable developers to write tests that interact with and assert things about current state of the DOM. These typically expose "selectors" (similar to [CSS selectors](https://developer.mozilla.org/en-US/docs/Learn/CSS/Building_blocks/Selectors)) for specifying an element(s) on the page, which you can then assert things about (e.g. is it visible?) or perform actions on (e.g. click it).

The selectors generally support some combination of searching by tag name (e.g. `div`), CSS class names (e.g. `.foo`), id (e.g. `#foo`), or arbitrary DOM attributes (e.g. `[role="button"]`). Sometimes they can also refine based on text content (e.g. `:text("exact match")` or `:contains("fuzzy match")`).

While useful, most of these existing APIs generally have downsides:
* They rely on React internals to implement the React component name matching.
* They often only support DOM.
* They do not do enough to discourage brittle test selectors from being written (e.g. `"ProfilerHeader NavLink label a"`).
  * Test ids provide an alternative to this but they [have scaled very poorly in practice](https://gist.github.com/bvaughn/d3c8b8842faf2ac2439bb11773a19cec#gistcomment-3966895).

A first-party React selector API could be adopted by these frameworks so they are no longer be renderer-specific, (although this might require imposing constraints on what the selector syntax supported). It could also make open source libraries like `[react-testing-library](https://github.com/testing-library/react-testing-library)` more powerful.

Specifically, the API proposed below has the following goals:
* Remove dependency on React internals that may change in future releases.
* Support all React renderers.
  * The examples in this doc are all DOM, but this API should support other renderers (e.g. React Native).
  * For example, this API would enable Jest to support full React Native e2e tests.
* Enable useful tests to be written with minimal knowledge of component implementation details.
  * Components opt-in by adding a `data-testname` attribute to host elements they wish to expose for testing. This enables significant refactoring flexibility (e.g. adding/moving/removing rendered elements) without breaking tests.
* Reduce the amount of test-specific annotations required in product code.
  * The only host elements requiring `data-testname` attributes are ones that tests interact with directly.
  * In many cases (e.g. clicking or verifying visibility) `data-testname` attributes aren’t even required.
* Support HOCs and other behavioral components without requiring pass-thru test id props.
  * e.g. a behavioral component like `LoginButton` can be combined with a generic `data-testname` attribute like “button” to select a specific host element, even if that element was rendered by a generic/shared React component like `Button`.
* Enable “sparse” selectors to be written without requiring the full path.
  * e.g., identifying the login button in the page header is often possible without specifying the full path to that element. This allows intermediate components to be added/moved/removed without breaking tests.
* Avoid the constraint of globally unique ids.
  * Although `data-testname` attributes could be unique, this design does not require uniqueness.
* Enable additional behavior (or constraints) to be implemented on top of React selectors.
  * e.g. pseudo-selectors like `:contains` and `:text` can be built on top of this API ¹
  * e.g. syntax sugar (`"Navigation Link#link:text("Contact")"`) could be supported by this API ¹

¹ Examples below show how this could be implemented.

# Detailed design

## Implementation

The APIs below would likely be implemented as (optional) methods on the HostConfig which get eliminated in non-test builds. The DOM HostConfig could use general browser APIs like `getClientBoundingRect` and `IntersectionObserver` to implement most of the behavior described.

If browser APIs prove not to be precise enough, we could detect that we’re in a E2E environment and call a global object passing a DOM node expecting a response, e.g.
```
if (typeof __E2E_OBSERVE_RECT__ === "function") {
  __E2E_OBSERVE_RECT__(node, ...);
}
```

This would not be considered public API but rather a contract between React and an E2E testing framework.

I propose we add the following methods to a special **testing** build of renderer . The names proposed in this RFC are not final, although the expected behavior is.

# Creating selectors

The core API methods below work with an array of “selectors”. A selector can be any of the following:
* Component type (`React$AbstractComponent<empty, mixed>`)
* Role (string, e.g. “header” would match an `<h1>` element)
* Text contents (string)
* Test ID (`data-testname` attribute)
* Pseudo selector (array of selectors used to identify an element without actually selecting its subtree)

The methods below are used to create selectors which can then be passed to other methods (like `findAllNodes()`) to specify which element(s) you want to operate on.Component

## `createComponentSelector()`

Matches host elements rendered by a specific React component type.

```js
createComponentSelector(ExampleComponent);
```

## `createRoleSelector()`

Matches host elements with a specific accessibility role. Matches can be explicit (e.g. `role="button"`) or implicit (e.g. `<button>`).

```js
createRoleSelector("button");
```

## `createTextSelector()`

Matches host elements containing the specified text string.

```js
createTextSelector("Exaple text");
```

## `createTestNameSelector()`

Matches host elements with a specific test name data attribute (e.g. `data-testname="LoginFormButton"`).

```js
createTestNameSelector('LoginFormButton')
```

## `createHasPseudoClassSelector()`

Inspired by CSS [pseudo-classes](https://developer.mozilla.org/en-US/docs/Web/CSS/Pseudo-classes), this selector matches host elements with subtrees that match the sub-selector array.

For example, given the following app:
```html
<div>
  <article>
    <h1>Should match</h1>
    <p>
      <button>Like</button> <!-- We want to select this one -->
    </p>
  </article>
  <article>
    <h1>Should not match</h1>
    <p>
      <button>Like</button>
    </p>
  </article>
</div>
```

Then we could use `createHasPseudoClassSelector` to select the "like" button inside of the _first_ list item:
```js
[
  createRoleSelector('article'),
  createHasPseudoClassSelector([
    createRoleSelector('heading'),
    createTextSelector('Should match'),
  ]),
  createRoleSelector('button'),
]
```

# Primary Core API

## `findAllNodes()`

Finds all host elements (e.g. `HTMLElement`) within a host subtree that match the specified selector criteria.

```
function findAllNodes(
  hostRoot: Instance,
  selectors: Array<Selector>,
): Array<Instance>
```

### How does it work?

* This method starts searching at the specified root, which must be either:
  * A host parent node above the React container (e.g. `document.body`) in which case, React will traverse the native tree until it finds the first React container, or...
  * A React container, or...
  * A React-rendered host instance with a `data-testname` attribute (e.g. a match from an earlier `findAllNodes` query).
* Starting from the root (or React container) React traverses the internal Fiber tree ¹ looking for all paths matching the specified selector.
  * While traversing the tree, each node will be compared to the current selector index.
      * If the current selector doesn’t match, the tree traversal continues.
      * If the current selector matches, the selector index is advanced.
  * Once all selector criteria have been satisfied, the remaining host elements will be returned.
* (If the selector array is empty, the root host instance’s fiber will be used.)

¹ Traversing the Fiber tree (instead of the host tree) provides several benefits:

* Easier to implement and share cross-renderer logic.
* Portals (e.g. tooltips) can be matched even if they are in a different host tree.

### What does it return?

* An array of host elements that match the specified criteria. (This array will be empty if no matches are found.)

## `getFindAllNodesFailureDescription()` ¹

Prints the names of components within a matched subtree that directly renders host elements.

```
function getFindAllNodesFailureDescription(
  hostRoot: Instance,
  selectors: Array<Selector>,
): string | null
```

¹ I believe this API would help with adoption, but **it is a non-essential part of the RFC** and could be dropped. I had initially planned for `findAllNodes` to throw an error with this description when no matches were found, but there are cases where no matches is *expected* (e.g. `expect('...').not.toBePresent()`). An e2e framework could try/catch these cases to suppress the error from end users, but this would also complicate chained selector implementation (see example below).

### How does it work?

* This methods processes selectors in the same way as `findAllNodes`.
* If no matching host elements can be found, it returns a string describing why.

### What does it return?

* If the method was unable to match all of the specified selectors, it returns a string specifying which selectors were matched and which selectors were not matched.
* If the selector finds at least one matching host element, this method returns `null`.

# Secondary API

The `findAllNodes` API is powerful because it gives you full access to the DOM node. However, it also requires low-level interaction with that DOM node. Certain higher level interactions are broadly useful.

For example: how much space does a component take up in its parent container? The reason this can be thought of as part of the public API is because you can measure it using a wrapper element’s bounding box. (In other words, it can be measured without compromising the component’s technical implementation details.) Furthermore if a component’s size changes, it’s likely part of an intentional visual change.

## `findBoundingRects()`

For all React components within a host subtree that match the specified selector criteria, return a set of bounding boxes that covers the bounds of the nearest (shallowed) Host Instances within those trees.

```
function findBoundingRects(
  hostRoot: Instance,
  selectors: Array<Selector>,
): Array<Rect>
```

### How does it work?

* This methods processes selectors in the same way as `findAllNodes`.
* Within each matching path:
  * Find the first Host Instances in the matched component. (Might be several for Fragments.)
  * For each Host Instance, call a `getBoundingClientRect` Host Config method. Add the result to a list of matched boxes.
  * Optionally merge adjacent boxes and exclude boxes that are fully covered by other boxes. (This ensure less reliance on implementation details like how many nodes make up the tree.)
* Return the list of matches bounds.

### What does it return?

* An array of bounding boxes relative to the viewport. E.g. `{ x: number, y: number, width: number, height: number }` (This array will be empty if no matches are found.) 

### What can this API be used for?

* Clicking or hovering over the center of a Component.
* Take a snapshot of the Component for screen shot tests.
* Assert on the maximum size a component is expected to take up.
* Scrolling the Component into view.
  * This might only be enough for scrolling the outer most scroll. It’s not enough for scrolling inner components. However, inner component’s scrolling may be implemented with other techniques than native scrolling so exposing a method for this would leak implementation details. For this use case `data-testname` on the scrollable container might be appropriate.

## `observeVisibleRects()`

For all React components within a host subtree that match the specified selector criteria, observe if it’s bounding rect is visible in the viewport and is not occluded.

```
function observeVisibleRects(
  hostRoot: Instance,
  selectors: Array<Selector>,
  callback: (intersectingRects: Array<{ ratio: number, rect: Rect }>) => void,
  options: IntersectionObserverOptions,
): { disconnect(): void }
```

### How does it work?

* Create a new [`IntersectionObserver`](https://developer.mozilla.org/en-US/docs/Web/API/IntersectionObserver/) or equivalent HostConfig specific API, passing a `callback` and `options` argument to it.
  *  Note: Host Config implementation needs to convert each entry into an object containing only `{ ratio, rect }`, otherwise this callback works the same as the DOM version.
* This methods processes selectors in the same way as `findAllNodes`.
* Within each matching path:
  * Find the first Host Instances in the matched component. (Might be several for Fragments.)
  * For each Host Instance, call the `observe(intersectionObserver, instance)` Host Config method to add it to the list to be observed.
* Attach an internal (for test builds only) callback to the React reconciler that is called every time React commits a new update.
  * When this callback is invoked, disconnect the Observer and start over from the top to attach a new one. This makes this observation live updating. This prevents accidentally relying on implementation details such as if a DOM node is reused or remounted.
      * Optionally observe/disconnect only the added/removed Host Instances.

### What does it return?

* An object containing a `disconnect()` method. When this method is called, we disconnect the `IntersectionObserver` and disconnect the callback in the React reconciler.

### What can this API be used for?

* Waiting for a Component to become visible before doing further operations.
* Asserting that a Component is already visible.

### Why is this API necessary?

* Ideally `findBoundingRects` would be enough for this but these APIs don’t give access to where in the DOM this component is, nor really all the other things that might obscuring the bounding rect. This lets us solve this use case without exposing more implementation details and we can build in details such as if it’s obscured by something virtual (e.g. if an ART component is covered by another ART component within a canvas).

## `focusWithin()`

For all React components within a host subtree that match the specified selector criteria, set focus within the first focusable Host Instance (as if you started before this component in the tree and moved focus forwards one step).

```
function focusWithin(
  hostRoot: Instance,
  selectors: Array<Selector>,
): boolean
```

The internal structure of a node is an implementation detail. However, you can start from the outside of a component and move forward (e.g. hit tab) to focus within a component. This has behavior that is defined, and so it can be thought of as part of the public API fo the component.

With the Focus Selectors API used for a11y we might be able to even expose more specific capabilities for the outside of a component move focus into a child.

### How does it work?

* This methods processes selectors in the same way as `findAllNodes`.
* Within each matching path:
  * Find the first Host Instances in the matched component. (Might be several for Fragments.)
  * For each Host Instance:
      * If this Host Instance is considered Focusable by calling the Host Config `isFocusable(instance)`, call the Host Config `focus(instance)`.
      * Return `true` once we’ve found a focusable host instance.
* If we didn’t find a focusable host instance, return `false`.

### What does it return?

* A boolean indicating if something focusable was found and focus was set on it.

### What can this API be used for?

* Setting focus on the canonical part of a Component such as text input.

# Usage examples

## `findAllNodes()`

Here is a small app with a few components. We’ll use this app for the examples below.

```
// App.react.js
export default function App() {
  return (
    <main data-testname="main" `role``=``"main"`>
      <Header />
    </main>
  );
}

// Header.react.js
export default function Header({ title }) {
  return (
    <Fragment>
      <PageTitle title="Example" />
      <Navigation />
    </Fragment>
  );
}

// PageTitle.react.js
export default function PageTitle({ title }) {
  return <title>{title}</title>;
}

// Navigation.react.js
export default function Navigation() {
  return (
    <nav role="navigation" aria-label="Main">
      <SearchInput />
      <ul data-testname="list">
        <li><Link label="Home" /></li>
        <li><Link label="About" /></li>
        <li><Link label="Contact" /></li>
      </ul>
    </nav>
  );
}

// SearchInput.react.js
export default function SearchInput() {
  return <input data-testname="search" />;
}

// Link.react.js
export default function Link({ label, ...rest }) {
  return (
    <a data-testname="link" {...rest}>
      {label}
    </a>
  );
}
```

The app above would render HTML like this:

```
<main>                                               <!-- App -->
  <title>Example</title>                             <!-- PageTitle -->
  <nav `role``=``"navigation"`>                            <!-- Navigation -->
    <input data-testname="search" />                <!-- SearchInput -->
    <ul>                                             <!-- Navigation -->
      <li>                                           <!-- Navigation -->
        <a data-testname="link">Home</a>            <!-- Link -->
      </li>
      <li>                                           <!-- Navigation -->
        <a data-testname="link">About</a>           <!-- Link -->
      </li>
      <li>                                           <!-- Navigation -->
        <a data-testname="link">Contact</a>         <!-- Link -->
      </li>
    </ul>
  </nav>
</main>
```

### Verifying that an element was rendered

Here is how a test might verify that the `Navigation` menu was rendered:

```
await expect('Navigation#link').toBeVisible();
```

The selector above could be parsed and converted to use the `findAllNodes` API:

```
import {
  createComponentSelector,
  createTestNameSelector,
  findAllNodes,
} from "react-dom/testing";

const elements = findAllNodes(
  document.body,
  [
    createComponentSelector(require("Navigation.react")),
    createTestNameSelector("link"),
  ],
);

if (elements.length === 0) {
  // Throw error
}
```

Additional processing could then be done with the returned element to assert visibility (e.g. `expect(element).toBeVisible()`)

### Verifying the number of rendered elements

Here is how a test might verify that three links are rendered by `Navigation`:

```
await expect('Navigation Link#link').toBePresentCount(3);
```

The selector above could be parsed and converted to use the `findAllNodes` API:

```
const elements = findAllNodes(
  document.body,
  [
    createComponentSelector(require("Navigation.react")),
    createComponentSelector(require("Link.react")),
    createTestNameSelector("link"),
  ],
  "list"
);
```

Additional processing could then be done on the matched elements (e.g. `toBePresentCount`)

Note that given the application described above, the following selectors would also work:

* ✓ `App#link`
* ✓ `App Navigation#link`
* ✓ `Navigation#link`
* ✓ `App Navigation Link#link`

### Implementing advanced pseudo-selectors (e.g. `:contains`, `:text`)

Here is how a test might use additional pseudo-selectors to click the “Contact” link rendered by `Navigation`:

```
await wwwPage.click('Navigation Link#link:text("Contact")');
```

The selector above could be parsed and converted to use the `findAllNodes` API:

```
const elements = findAllNodes(
  document.body,
  [
    createComponentSelector(require("Navigation.react")),
    createHasPsuedoClassSelector([
      createComponentSelector(require("Link.react")),
      createTestNameSelector("link"),
      createTextSelector("Contact"),
    ]),
  ],
  "list"
);
```

### Portals

Portals cause divergence between the React tree and the host tree. Consider the following demo application:

```
function Parent() {
  return (
    <div>
      <Child />
    </div>
  );
}

function Child() {
  return (
    <div>
      <Grandchild />
    </div>
  );
}

function Grandchild() {
  return createPortal(
    <div data-testname="portal" />,
    document.getElementById("portal")
  );
}
```

This app would be seen by React as the following tree:

```
Parent
  <div>
    Child
      <div>
        Grandchild
          <div data-testname="portal" />
```

However it might be rendered into the DOM as the following:

```
<body>
  <div id="root">
    <div> <!-- owned by parent -->
      <div> <!-- owned by child -->
  <div id="portal">
    <div data-testname="portal"/> <!-- owned by grandchild -->
```

Because `findAllNodes` traverses the fiber tree, it will be able to find the portal even if given the “root” container. It should be possible then for all of the following queries to find matching host elements:

* ✓ `Parent#portal`
* ✓ `Parent Child#portal`
* ✓ `Parent Child Grandchild#portal`
* ✓ `Child#portal`
* ✓ `Child Grandchild#portal`
* ✓ `Grandchild#portal`

### Render props

Render props are another interesting edge case for an API like this. Consider the following demo application:

```
function Parent() {
  const render = () => (
    <div key="parent" data-testname="parent" />
  );
  return <Child render={render} />;
}

function Child({ render }) {
  return (
    <div key="child" data-testname="child">
      {render()}
    </div>
  );
}
```

This app would be seen by React as the following tree:

```
Parent
  Child
    <div data-testname="child"/>
      <div data-testname="parent"/>
```

And it would be rendered into the DOM as the following:

```
<div id="root">
  <div data-testname="child">
    <div data-testname="parent">
```

Because `findAllNodes` traverses the fiber tree, the following selectors will correctly identify test host nodes:

* ✓ `Parent#parent`
* ✓ `Parent Child#child`
* ✓ `Child#child`

## `getFindAllNodesFailureDescription`

### Debugging failed selectors

If a test fails because an expected match was not found, it is useful to provide a developer with actionable feedback. Here is a selector that would not find a match given the example app above:

```
await wwwPage.click('Header PageTitle Link#link');
```

In this case, the e2e framework knows that a match was expected (because the `click` action was used) so it should provide the user with an actionable error message. `getFindAllNodesFailureDescription` can help with this.

```
getFindAllNodesFailureDescription(document.body, [
  createComponentSelector(require("Header.react")),
  createComponentSelector(require("PageTitle.react")),
  createComponentSelector(require("Link.react")),
  createTestNameSelector("link"),
]);
```

In this case the returned description might be something like the following (depending on the `displayName` values for the specified React components):

```
``findAllNodes` was able to match part of the selector:
  `Header` > `PageTitle
`
No matching component was found for:
  `Link``
```

## `findBoundingRects()`

Test may want to take screenshot snapshots of a component:

```
await expect('Navigation.react').toMatchScreenshot();
```

We could use the `findBoundingRects` API for this:

```
const rects = findBoundingRects(document.body, [
  createComponentSelector(require("Navigation.react")),
]);
```

User Puppeteer as an example then, we might do:

```
const rect = merge(rects);

await page.screenshot({
  clip: {
    x: rect.left,
    y: rect.top,
    width: rect.width,
    height: rect.height
  }
});
```

## `observeVisibleRects()`

### Verifying that an element was rendered

The `findAllNodes` example above could be rewritten to use the `observeVisibleRects` API instead. This change would likely encompass a larger host subtree but would also eliminate the need for a `data-testname`. For example:

```
await expect('Navigation').toBeVisible();
```

The selector above could be parsed and converted to use the `observeVisibleRects` API:

```
let matched = false;

const callback = recs => {
  recs.forEach(({ ratio: number, rect: Rect }) => {
    // Some business logic to approve or reject based on the ratio, e.g.
    if (ratio > 0.5) {
      matched = true;
    }
  });
};

const { disconnect } = observeVisibleRects(
  document.body,
  [createComponentSelector(require("Navigation.react"))],
  callback
);

// Optionally wait some amount of time until the element becomes visible.

disconnect();

if (!matched) {
  // Throw error
}
```

## `focusWithin()`

### Clicking on a link

Given the above example code, we could click on the first link within the `Navigation` menu using the `focusWithint` API:

```
focusWithin(document.body, [
  createComponentSelector(require("Navigation.react")),
]);

// document.activeElement is now the 1st focusable host instance in Navigation.
```

User Puppeteer as an example then, we might do:

```
`const`` element ``=`` await page``.``evaluateHandle``(()`` ``=>`` document``.``activeElement``);
``await element.click();`
```

### Entering text into a search input

Given the above example code, we could use the `focusWithint` API to search:

```
focusWithin(document.body, [
  createComponentSelector(require("SearchInput.react")),
]);

// document.activeElement is now the 1st focusable host instance in SearchInput.
```

User Puppeteer as an example then, we might do:

```
await page.keyboard.type("Type this into the focused element...");
```

# Drawbacks

* Supporting all renderers will require a more constrained API than if we targeted only one renderer (e.g. DOM).

# Alternatives

* Adapter strategy (like Enzyome uses). This has proven increasing hard to maintain as React releases new updates.
* Accessibility focused strategy (like Testing Library uses). TL shares a lot of similarities with this test selector proposal. This proposal offers a few advantages though, including better support for portals and a unified testing API to be adopted by any React renderer.

# Adoption strategy

* Work with library maintainers to adopt this new API.
* Adoption beyond this is optional and can move at its own pace.

Note that to use the test selector API at scale, you might want to use a Webpack alias (or equivalent) when building your test target:
```js
module.exports = {
  // ...
  resolve: {
    // ...
    alias: {
      // ...
      'react-dom$': 'react-dom/testing',
    },
    // ...
  },
  // ...
};
```

# How we teach this

Much of the above RFC (usage examples) could be converted to user-facing documentation. We will probably want to write an addition introduction explaining ways to structure tests, and testing methodology.

# Unresolved questions

* What are the best ways to require component definitions to passed to `createComponentSelector`?
* Will existing test libraries (other than potentially Jest) adopt this new API?
* More (to be uncovered by this RFC?)