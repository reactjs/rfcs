- Start Date: 2019-04-30
- RFC PR: (leave this empty)
- React Issue: (leave this empty)


# Summary

The purpose of this proposal is to allow developers to create custom context 
 Provider using hooks for state management. It's sort of a merge of 
 *FunctionalComponent* and *ContextProvider*.


# Basic example

This example shows how `context.createProvider(...)` could be used to create
a library that gives the screen height and width.

```js
import {
  createContext,
  useContext,
  useEffect,
  useMarkChangedBits,
  useState
} from "react";

const WIDTH_BITS = 0b0100;
const HEIGHT_BITS = 0b1000;

const getScreenSize = () => ({
  width: window.innerWidth,
  height: window.innerHeight
});

const INITIAL_CONTEXT_VALUE = getScreenSize();

const context = createContext();

export const ScreenProvider = context.createProvider(function ScreenProvider() {
  const [screenSize, setScreenSize] = useState(INITIAL_CONTEXT_VALUE);

  useEffect(() => {
    const handler = () => {
      setScreenSize(getScreenSize());
    };
    window.addEventListener("resize", handler);
    return () => {
      window.removeEventListener("resize", handler);
    };
  }, [setScreenSize]);

  useMarkChangedBits(WIDTH_BITS, [screenSize.width]);
  useMarkChangedBits(HEIGHT_BITS, [screenSize.height]);

  return screenSize;
});

export const useWidth = () => {
  const { width } = useContext(context, WIDTH_BITS);
  return width;
};

export const useHeight = () => {
  const { height } = useContext(context, HEIGHT_BITS);
  return height;
};
```


# Motivation

One of the benefits of React Hooks is to reduce the size of the virtual tree.

The first motivation is to apply this benefit to Context's Providers.

After writing the example, it also emerged that this proposal can simplify how
 the "changed bits" of a context value are computed.


# Detailed design

## Create a provider

A new function is added by this proposal:
 `context.createProvider(computeProvidedValue: Function): Provider`

The argument callback `computeProvidedValue` is similar to a Functional
 Component in that it takes props as its arguments, and can call hooks, but 
 returns the new context value instead of a React element.

## Hooks to set the changed bits

- `useMarkChangedBits(bits, [deps])`<br />
  Marks (set but cannot unset) the changed bits of the currently generated 
  context value.<br />
  It can be used with deps:
  `useMarkChangedBits(WIDTH_BITS, [screenSize.width]);`<br />
  Or it can be used with a dynamic `bits` argument:
  `useMarkChangedBits(changed ? 0b01 : 0);`

- `useMemoProvidedValue(createValue, [deps], bits)`<br />
  It merges the logic of `useMemo` with `useMarkChangedBits`, to set the 
  changed bits only if `useMemo` returns a different value.


## Changed bits without hooks

If no hooks are used to set the changed bits, React will compare the newly 
 provided value with the previous one, and set either all bits or no bits 
 changed.


# Drawbacks

- An implementation will likely require a new React element type, it doesn't 
  seem possible to reuse `FunctionComponent` nor `ContextProvider`. It will 
  add complexity to React's implementation. And tools (like react-devtool) would
  not be immediately compatible.

- `createProvider` is most likely to be used by third-party libraries. For 
  libraries to adopt it, they would need applications to upgrade to the latest 
  version of React (like it happened for hooks).


# Alternatives

No alternatives proposed. This proposal doesn't provide any *new* feature, and 
 `Context.Provider` is still usable.

# Adoption strategy

The proposal doesn't break any existing feature, and is compatible with 
 existing Context `Consumer` and `useContext`.
 
The implementation of the proposal is likely to add a new fiber type. Tools 
 will have to integrate with it (for example react-devtool).

*Changed bits* in `useContext` hooks is still unstable. It would be best to 
 stabilize it before releasing this proposal.

# How we teach this

This proposal relies a lot on existing notions: functional components, hooks,
 contexts. Documentation of the new functions and integration with 
 react-devtool should be enough.

# Unresolved questions

- The naming of the hooks, especially `useMarkChangedBits`.
- Should the proposal provide a function `markChangedBits(bits)`, like the hook
  (without the deps), but can be used in conditional branches or loops.
- The hook `useMemoProvidedValue` can be created in user land, is it worth 
  adding it to the proposal.
- How to prevent the hooks `useMarkChangedBits` and `useMemoProvidedValue` to
  be used outside of `createProvider`.
