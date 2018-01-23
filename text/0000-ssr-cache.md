- Start Date: 2018-01-23
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Add hooks to ReactDOMServer to support caching

[See the original discussion here](https://github.com/facebook/react/issues/11670)

# Basic example

```js
import { renderToString } from 'react-dom/server';

// No cache strategies
renderToString(<App ... />);

// Using cache strategy
renderToString(<App ... />, { cacheStrategy: new ExampleCacheStrategy() });
```

# Motivation

Why are we doing this? What use cases does it support? What is the expected
outcome?

* Improve SSR render performance

# Detailed design

##### API design goals:

1. SSR caching is performed at the `frame` level (of the ReactPartialRenderer), where each frame represents a react element
   * only complete/closed frames can be cached (this greatly simplifies the SSR cache API)

2. SSR caching is performed by a `CacheStrategy` implementation
   * cache strategies are passed as `options` to `renderToString()` and `renderToStream()`

3. Cache strategies should be capable using either react components or configuration to enable caching.


4. Beyond supporting simple per-component caching, a cache strategy _should be able to support_ `templates`, where a component can be rendered to become a (cached) template, and templates can then have content injected - this has the potential to _drastically_ improve SSR performance.


### CacheStrategy API

```js

interface CacheStrategy {

  /**
   * Gets the cache strategy state for a component.
   *
   * ReactPartialRenderer hook: resolveElement (called during every resolveElement invocation)
   *
   * @param component
   * @param props
   * @param context
   * @returns {*} if undefined is returned, the cache strategy render method will not be invoked for this component.
   */
  getCacheState(component: ReactNode, props: Object, context: Object): any,

  /**
   * Renders an element using a cache strategy.
   *
   * ReactPartialRenderer hook: renderFrame (called when rendering a frame that has assigned 'cacheState')
   *
   * @param element to render
   * @param context to use for rendering
   * @param cacheState the state returned by the getCacheState() method
   * @param renderUtils to simplify rendering of cached component
   * @returns {string} the rendered component
   */
  render(element: ReactElement, context: Object, cacheState: mixed, renderUtils: CacheRenderUtils): string,
}

```

#### Cache Strategy Notes

##### #getCacheState()
 * Determines if a component supports caching, and returns component specific cache state
 * **Hook** must be called by the `ReactPartialRenderer` every time an element is resolved
 * Returning `undefined` indicates that the component does **not** support caching


##### #render()
 * handles rendering for a component that supports caching
 * **Hook** must be called by `ReactPartialRenderer` when rendering a frame that has `cacheState`
 * receives the `cacheState` returned by `getCacheState()`
 * receives `renderUtils`, utility methods for rendering cached components that abstracts renderer internals

### CacheRendererUtils

```js

// Utility methods for rendering that abstract renderer internals
type CacheRenderUtils = {

  /**
   * Renders the current frame element and all its children, allowing props to be overridden.
   *
   * @param props
   * @param context
   * @returns {string} the rendered element output
   */
  renderCurrentElement: (props?: Object, context?: Object) => string,

  /**
   * Renders the provided element and all its children.
   *
   * @param element
   * @param context
   * @param domNamespace
   * @returns {string} the rendered element output
   */
  renderElement: (element: ReactElement, context?: Object, domNamespace?: string) => string,

  /**
   * Logs a warning if the base context is modified during the provided render function.
   *
   * NOTE: This only logs warning messages in development.
   *
   * @param baseContext the expected context throughout the entire render method
   * @param render method
   * @param [messageSuffix] {string} the message to log
   * @returns {string} the render output
   */
  warnIfRenderModifiesContext: (baseContext: Object, render: (ctx: Object) => string, messageSuffix?: string) => string,
};
```

#### Render Util Notes

##### #renderCurrentElement()
 * renders the current element, allowing props and context to be modified

##### #renderElement()
 * renders the provided element
 * allows arbitrary elements to be rendered by cache strategies
 * this enables cache strategies to be created that supports injecting element(s) into cached content (aka, a template)
   * this method can be used to render the element(s) that will be injected into a template

##### #warnIfRenderModifiesContext()
 * enables a cache strategy to determine if the `context` is modified while rendering an element (and all its children)
 * this is useful for a cache strategy that supports injecting element(s) into cached content (aka, a template)
   * if the context is changed when rendering a template, injected elements may not render consistently
     on the server and client
   * this method can be used to wrap `renderCurrentElement` or `renderElement` and log a warning if rendering on server/client are at risk of being inconsistent.


I've created a [proof of concept you can view in this forked branch](https://github.com/adam-26/react/commits/ssr-hooks). It includes an example `CacheStrategy` implementation and basic app in the [SSR plugins fixture](https://github.com/adam-26/react/tree/ssr-hooks/fixtures/ssrPlugins).

The proof of concept app includes 3 approaches to caching:
1. Using a `static getCacheKey = (props) => 'key';` method on any component
2. Using a `<Cache ...>` component
3. Using a `<CacheTemplate>` component to create templates from components that support injecting content into the cached template.


### Template Implementation Example

IMPORTANT: The `Template` is an example of a hook implementation, it is **NOT** part of the SSR cache feature.

```js
<Template
  cacheKey={()=>'header'}
  component={Header}
  templateProps={ /* passed to component, will be cached as part of the template*/ }
  injectedProps={ avatar: Avatar /* passed to component, not cached */ } />
```
The `Header` can be rendered as a template that allows the `avatar` prop to be injected each render. This prevents the need to render the `Header` on each request.

The same approach could be applied to more complex components, such as a `<Product ... />` or `<Comments .../>` component, greatly reducing SSR time.

`RenderUtils` can be used to simplify `template` support.

##### Templates - and the problems of context
On the first render (for a given cache key), the component is rendered to a template (where a template is a tokenized string) using a Placeholder that replaces each of the injectedProps (you can see an example of this here). The Placeholder acts as a token that is replaced by injected content for future renders utilizing the cached template.

The template output is cached and used to render all future Header components (with the same cache key), and the Placeholders in the template are replaced with the rendered output of the injectedProps. This prevents the Header from being processed with lifecycle methods on the server - while still being able to accept props that change on each render (as a side note, the injectedProps could also use cache strategies for rendering).

This minimizes the CPU bound activity of a server render and reduces server render time. In the proof of concept linked above the render time is reduced > 60% (conservative measure of very basic comparisons) using the template approach for a component that is comprised of a small number of child components, as the number of the Template component children increase the time saved should be even greater - instead of rendering the component a cached string is being used.

For this to work consistently for each render, its important that no middle components (components between the Template component and any injectedProps) change the context being used to render the template. If the context is not changed, then all template renders will be consistent (The Placeholder can be used to ensure no additional props are applied to any injectedProps within the template component). This is why the warnIfRenderModifiesContext method is defined in the CacheRenderUtils above, the method is needed to guarantee template renders are consistent.

This could be taken even further, and cached templates could be written to a cache server (mem-cache, redis, etc..) and re-used across server instances.


# Drawbacks

Why should we *not* do this? Please consider:

* SSR render process becomes slightly more complicated

* Hook implementations can only cache react markup, any side-effects need to be managed (or cached) by alternative means. For example, if the component creates/populates any application state (such as redux store data) then that application state would also need to be cached for future renders.


# Alternatives

Implemeting a SSR caching strategy in user space is not possible without exposed hooks.

The only other alternative to implement SSR cache is to fork and modify the react-dom/server code.

# Adoption strategy

It should be relatively simple to adopt, no breaking changes are implemented.

# How we teach this

Add documentation about the hooks to the reactjs website, write a blog entry.

Its use is optional, only for react-dom and has no impact on new developers.

# Unresolved questions

For a complete cache solution, a solution is required for also caching component side-effects (data).

If you consider react to be a 'view' engine, then this is entirely reasonable - implementing a compatible cache solution for application state and/or data is simple enough.
