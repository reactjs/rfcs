- Start Date: 2018-03-30
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Node.js is introducing [native support of ES2015 modules](https://nodejs.org/api/esm.html). This will be available in Node.js 10 which is marked as being released in April 2018 on the roadmap.

For brevity **ESM** will be used to refer to the native implementation of ES2015 modules in Node.js with `.mjs` files.

Due to significant differences in how JavaScript behaves in ESM vs. CommonJS code this is being implemented using a new `.mjs` file extension.

ESM imports require named exports to be known ahead of time. However the names of CommonJS exports are not known until runtime and CommonJS also allows the entire `module.exports` to be replaced with a non-object. As a result when an ESM module imports CommonJS code it can only do so using the `default` export. You cannot use named imports in ESM code to import named exports from CommonJS code.

React only exports a CommonJS module from the package. As a result named exports do not work.

I recommend we add a small ESM export to React to allow named exports from React to work in ESM code.

# Basic example

The following examples are the current behaviour of React in various contexts.

If you use React in CommonJS modules written in ES2015 you use it as such:

```js
const React = require('react');
class MyComponent extends React.Component {}
```

```js
const {Component} = require('react');
class MyComponent extends Component {}
```

If you write ES2015 modules but transpile to CommonJS using Babel, use Rollup, or use WebPack the following will work:

```js
// Default imports
import React from 'react';
class MyComponent extends React.Component {}
```

```js
// Named imports
import React, {Component} from 'react';
class MyComponent extends Component {}
```

If you write ES2015 modules and deliver them to the browser with `<script type="module">` you cannot use React without either making React a global script or using some extra step to convert the CommonJS React code into a ESM bundle. I have not personally investigated this option at the moment, but I believe this likely suffers from the same limitations as ESM.

If you write ESM code: ES2015 module code in a `.mjs` file run by Node with `--experimental-modules` enabled (or without the flag in the upcoming Node 10). The following will work:

```js
// Default imports
import React from 'react';
class MyComponent extends React.Component {}
```

However in ESM code currently the following will break, as Component will be undefined:

```js
// Named imports
import React, {Component} from 'react';
class MyComponent extends Component {}
```

This proposal asks that we change this behaviour so that the former works in ESM code, just like it works in transpiled/bundled code.

# Motivation

Officially the React documentation only documents the use of named imports in examples. In fact the ES2015 source code of React just does an `export default React;` without exporting any named exports. Named imports only work due to the CommonJS bundling and exclusion of a `"module"` variant.

However despite not being part of the official docs, named exports from React are used regularly in the React community.

Here is a small sample list of popular libraries in the React ecosystem using named imports in ES2015 modules. Links go to a random .js file in the library's official repo showing one of their components using named imports.

- [react-redux](https://github.com/reactjs/react-redux/blob/master/src/components/Provider.js)
- [redux-form](https://github.com/erikras/redux-form/blob/master/src/Form.js)
- [Downshift](https://github.com/paypal/downshift/blob/master/src/downshift.js)
- [react-intl](https://github.com/yahoo/react-intl/blob/master/src/components/message.js)
- [react-dnd](https://github.com/react-dnd/react-dnd/blob/master/packages/react-dnd/src/DragDropContext.js)
- [Create React App's default template](https://github.com/facebook/create-react-app/blob/next/packages/react-scripts/template/src/App.js) (and all users of *CRA* who just follow how the default *CRA* template imports React)

All of these are users of the non-spec transpiled/bundled version of ES2015 modules and work due to it's non-spec handling of CommonJS exports. However if any of these parts of the community decide they wish to use or at least support the use of native ESM code, currently they will have to refactor their entire codebase to use `React.Component` off a default import instead of using named imports

# Detailed design

Here are the requirements that the ESM implementation in Node.js place on us and it's behaviours we can take advantage of:

- ESM code **must** be inside of a `.mjs` file
- ESM code **must** use ES2015 `import` and `export` and cannot use `require`
- ESM code **must** declare all of its named exports in the `.mjs` file or they must come from another `.mjs` file exported using `export * from './somefile'`. (You likely could use `*` on a CommonJS module, but it would only export `default`)
- If the file being imported by an ESM `import` is a CommonJS module the entire module's exports object will be available as the `default` export, named exports will all be undefined
- A CommonJS module cannot `require` an ESM module, however it can asynchronously `await import('./somefile.mjs')` one
- A CommonJS module's `require` with no extension will look for `.js` files but will not look for `.mjs` files
- A ESM `import` without an extension will **first** look for an `.mjs` file and will only look for a `.js` if an ESM module is not found
- As a result if you place a CommonJS `module.js` and an ESM `module.mjs` next to it `require('./module')` will require the `module.js` and `import './module';` will import the `module.mjs`
- This behaviour is the same for the main `index.js`/`index.mjs` import of the package if you define `main` as `"main": "./index"` without the file extension
- ESM code may not use conditional imports, but CommonJS modules may
- The identifiers of all named exports from a ESM module must be defined ahead of time, **however** the value of those exports do **not** need to be defined in ESM code

Thus it should be sufficient to do the following:
- Change the `"main"` of react packages to be `"index"` instead of `"index.js"`
- Add an `index.mjs` that does an `import React from './index.js';` and `export default React;`
- In this file provide explicit exports of all the stuff on `React.*` that we wish to export

For example, `react/packages/react/npm/index.mjs` may look like this.

```js
import React from './index.js';

export const Children = React.Children;

export const createRef = React.createRef;
export const Component = React.Component;
export const PureComponent = React.PureComponent;

export const createContext = React.createContext;
export const forwardRef = React.forwardRef;

export const Fragment = React.Fragment;
export const StrictMode = React.StrictMode;
export const unstable_AsyncMode = React.unstable_AsyncMode;

export const createElement = React.createElement;
export const cloneElement = React.cloneElement;
export const createFactory = React.createFactory;
export const isValidElement = React.isValidElement;
export const version = React.version;

export const Component = React.Component;
export const PureComponent = React.PureComponent;

export default React;
```

# Drawbacks

I understand React recently reorganized [to use flat bundles](https://reactjs.org/blog/2017/12/15/improving-the-repository-infrastructure.html#compiling-flat-bundles) this may be a change that affects how all packages in the React monorepo are organized in order to support ES modules.

This change makes it possible to use React with ESM code in Node.js 10. However it does not help with loading React in `<script type="module">` code in browsers. Browsers only handle ES2015 imports, they do not understand CommonJS modules and as a result the ESM export will not be able to import the CommonJS bundles React builds.

However this may be acceptable for now as a transition period because browsers do not support the non-relative node style imports like importing `fbjs/lib/invariant` that React makes use of.

# Alternatives

## Drop support

Alternatively we could decide that we are not going to support named imports of React from ESM.

This will not be a breaking change. Use of .mjs and ESM is opt-in functionality. All packages using babel or a bundler to access node will keep working with the non-spec compliant handling of named exports from CommonJS modules.

The only requirement will be that if someone decides to switch to `.mjs` and opt-in to using ESM instead of transpiling/bundling, they will be required to refactor their entire codebase to use `React.Component` instead of named exports.

## ESM bundles

I understand that the community is still in flux regarding picking up ESM, for this reason I am proposing a workaround that I believe can be integrated into React with the least resistance.

However if the React team wishes it should also be possible to adopt ESM as part of the bundles themselves.

- The React source code may need to be double checked to ensure that it properly conforms to ESM and does not rely on any of the non-spec behaviours of the import->require transpilation.
- The source code of `packages/react/src/React.js` will need to be changed to both export named exports like `export {createRef};` and `export default` the default React object.
  - Alternatively, React.js could exclusively export named exports and an alternative to `index.js` could both `export * from './React';` and `import * as React from 'react'; export default React;` to get the `React` object as a free side effect.
- The build process will need to output `./esm/react.{production.min,development}.mjs` bundles in addition to the normal `./cjs/` bundles. I expect these will just be the Rollup bundle already being generated but without the module transpilation step.
- A `index.mjs` will sit at the root beside `index.js`, it will act the same as `index.js` but will export the `./esm` bundles using ES2015 exports instead of the `./cjs` bundles.
  - Unresolved: It is not actually possible to do the `if (process.env.NODE_ENV === 'production') {` conditional that `index.js` does in ESM, ES modules require that import is at the top of the file and does not permit conditional importing.

# Adoption strategy

Non-ESM users are unaffected by the changes to ESM handilng. Additionally it will sill be possible to use `React.Component` so this will not be a breaking change for current ESM users.

# How we teach this

If implemented this change will simply bring the behaviour of importing React from ESM code in line with the behaviour of importing React from transpiled/bundled code.

As a result it should not be necessary to teach any user how to use React in ESM. Though we may want to warn users that versions of React prior to the one we release this fix in do not export named exports when you require them from `.mjs` files.
