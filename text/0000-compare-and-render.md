- Start Date: 2018-10-28
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

During the reconciliation ReactDOM compares "old" rendered element, and a new one.
If type or key does not match - the old element will be removed, and new one will took it's place.

This makes stuff, like hot-reloading, a bit complex, and very flaky.

Everything could be solved by a dev-time ability to control tree reconciliation, via a "compare-control" hook inside React-DOM internals.
"compare-control" hook will be able to control the moment _type comparison_, telling React that
 types are still the same, even if not equal by a reference, and let React-Hot-Loader rely on
React internals, not to hack them.  

# Basic example
```js
// default behavior - pick A if A and B the same
React.compareControl( (typeA, typeB) => typeA === typeB ? typeA : null)

// threat any components with the same name as the same
React.compareControl( (typeA, typeB) => typeA.displayName === typeB.displayName)

```
# Motivation

This change would be able to solve some React-Hot-Loader issues:
1. [issue](https://github.com/gaearon/react-hot-loader/issues/304), `<Component />.type !== Component`, as long return type from createElement is a "Proxy".
2. [issue](https://github.com/gaearon/react-hot-loader/issues/1088), React-Hot-Loader is not _hooks-friendly_. The self-tree rendering is not working anymore.
3. [issue](https://github.com/gaearon/react-hot-loader/issues/1024), multiple logical issues, we cannot solve in a "user space"
4. (fact), React-Hot-Loader convert all SFC into Classes.

To prevent any future problem and enable a better HMR experience we need to change React.
This logic would be needed not only for React-Hot-Loader, but for [any other](https://github.com/facebook/create-react-app/pull/2304) "hot-react-replacement",
as least to mitigate side effects.

# Detailed design

During Hot Module Replacement the _type_ for a existing component changes, and this causes 
a whole tree remount.

React-Hot-Loader mitigates this by hacking into `createElement`, and always providing
the same _type_ for different component generations. The _type_ is always not equal to
the original type, and represents "Proxy" type, which is always a Class-based component.

To track "similarity" React-Hot-Loader performs a `register` operation, alering the code by babel plugin,
bounding _type_ to a file and a local variable name of the _type_ origin.

Unfortunately this operation could not handle types created in HOCs, or just not extracted as a 
top level variables.

To manage those "hidden" types React-Hot-Loader travers React-Tree by it'own, comparing
existing tree, with the one it renders itself.

`compare-control` hook is React alternative to this operation, idea was already proven
via integration to [Preact](https://github.com/developit/preact/pull/1120), which, hopefully, was not yet merged.

### API

`compare-control` should return false, if types are different, and new one should __replace__ the old.
`compare-control` should return true, if types are equal, and __new one__ should be rendered in the old place.

If _type_ has as instance, created using a old type - it should be preserved.

Results(after disabling most of React-Hot-Loader logic):
- hooks supported out of the box.
- SFC are used directly, and not wrapped by Class components.
- No need of custom tree traversal and rendering.
- React-Hot-Loader would be able to perform "hot-patching" inside _comparison_ function([like this](https://github.com/gaearon/react-hot-loader/blob/acb748b4e410f3b72cd1079f741223f3606f2f23/src/reconciler/hotReplacementRender.js#L428))

# Drawbacks

Why should we *not* do this? Please consider:

- this is only for the React-Hot-Loader
- some libraries still need a custom tree traversal, and it will solve the same problem.

# Alternatives

There is no other future-proof way

# Adoption strategy

Not to be used by developers.

# How we teach this

This is not a public feature

# Unresolved questions

- Ideally on the type change React should recreate an instance, with all the data preserved.
This is quite dangerous, and could be better solved by hooks.

- SFC are wrapped by Components for ability to _forceUpdate_ them. How to update _naked_ SFC, especially _memo_-ized.

- It would be nice to have _before rendering_ hook(componentWillUpdate), or have an access to instance, to apply instance-bound changes.
Right now we have to wrap _render_ method, and that's [breaking Dev Tools](https://github.com/facebook/react-devtools/pull/1191) 