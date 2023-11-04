- Start Date: 2022-08-25
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Implement an easy-to-use global state hook with unique keys and batch updates. No need for createContext, provider, useContext, it is as simple as using useState, the difference is that there is a unique key.

# Basic example

#### 1.First set a unique `key` for the global state, and can also set an initial value.

```tsx
const initStep = 1;
const key = 'global-step';
const [step, setStep] = useGlobalState(key, initStep);
```
##### Or encapsulate a `useGlobalStep` twice

```tsx
const useGlobalStep = (initStep: number = 0) => {
  const key = 'global-step';
  useGlobalState(key, initStep);
}
```

#### 2.Introduce `useGlobalState` in other components or pages and use the same `key`.

```tsx
const PageA = () => {
  const key = 'global-step';
  const [step, setStep] = useGlobalState(key, 1);
  return <div>
    {step}
    <Button onClick={() => setStep(step + 1)}>step + 1</Button>
  </div>
}

const PageB = () => {
  const key = 'global-step';
  const [step, setStep] = useGlobalState(key, 1);
  return <div>
    {step}
    <Button onClick={() => setStep(step - 1)}>step - 1</Button>
  </div>
}
```

##### Or use the secondary encapsulated `useGlobalStep`.

```tsx
const PageA = () => {
  const [step, setStep] = useGlobalStep();
  return <div>
    {step}
    <Button onClick={() => setStep(step + 1)}>step + 1</Button>
  </div>
}

const PageB = () => {
  const [step, setStep] = useGlobalStep();
  return <div>
    {step}
    <Button onClick={() => setStep(step - 1)}>step - 1</Button>
  </div>
}
```

# Motivation

Context is easy to be abused. In actual development, developers may put a lot of things in it, which makes management difficult and performance difficult to optimize. The design of useGlobalState is consistent with useState, which can avoid this problem.

The current react needs to use the context api to achieve state sharing. This may not be suitable and easy in terms of semantics and usage.

It is applied to pages or components that require the same state, and does not require context and provider reuse scenarios.

It can be implemented using unique keys and batch updates, which is much simpler, as easy as using useState, the only difference is an additional key parameter.

# Detailed design

![architecture diagram](https://bojuematerial-prudcut-public.oss-cn-guangzhou.aliyuncs.com/fdsafdsfdggfhfhgfgds.png?t=1)

# Drawbacks
  
- keys in use may conflict.
- when several places are used at the same time, the setting of the initial value may be confusing to the user.
- the user does not know the difference with the context.

# Alternatives

The saved state needs to be cleaned up immediately, otherwise it will cause a memory leak. Consider adding a throw error when the key is duplicated.

# Adoption strategy

This is a new api, the usage method is similar to useState, it does not require too much learning cost, and there is no historical burden.

# How we teach this

Tell the user that useGlobal has no provider, cannot be reused, and how to use it differently from context.
