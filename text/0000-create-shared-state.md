- Start Date: 2019-11-08
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

I would like to add a single function which allows sharing data between components with hooks. This could be used instead of Context API when scoping or cascading to single component structure branch is not needed. In many cases this leads to more simple implementation compared to using Context API. `createSharedState` creates a hook very similar to useState but with sync state across hook instances. Setting a value in one component will result re-rendering every component which uses the same hook. The state is preserved even if all hooks become unmounted.

## Side-by-side comparison with Context API
![image](https://user-images.githubusercontent.com/3163392/68534701-aedc9e00-0337-11ea-89c3-7eed540f23cd.png)

# Basic example

I suggest having an API such as this:

```jsx
import { createSharedState } from 'react';

const useTheme = createSharedState('light');

function App {
  return (
    <Toolbar />
    <ThemeSwitch />
  );
}

function Toolbar() {
  return (
    <div>
      <ThemedButton />
    </div>
  );
}

function ThemedButton() {
  const [theme] = useTheme();

  return <Button theme={theme} />;
}

function ThemeSwitch {
  const [theme, setTheme] = useTheme();

  return (
    <Button
      onClick={() => setTheme(theme === 'dark' ? 'light' : 'dark')}
    >
      {theme}
    </Button>
  );
}
```

For full working example please check https://codesandbox.io/s/react-create-shared-state-demo-9s9ui.

# Motivation

`createSharedState` results with a cleaner codebase with easily reusable shared state. Using this function will eliminated not necessary re-renders caused by top-level `Provider` in case of syncing data between nested components on different branches of component tree.

In React community many projects are already created with very similar idea:
- https://github.com/pie6k/hooksy
- https://github.com/kelp404/react-hooks-shared-state
- https://github.com/philippguertler/react-hook-shared-state
- https://github.com/dimapaloskin/react-shared-hooks
- https://github.com/atlassian/react-sweet-state
- https://github.com/paul-phan/global-state-hook
- https://github.com/donavon/use-persisted-state

Even though the solution can be easily created in user-land my motivation is the following to add it to core API:
- Most of developers would not know about it's existence if it's not mentioned in React documentation. Usually training sessions, blogs post about React are based on that React core API documentation and tutorial
- The code is very small and simple. It will not increase the bundle size much

# Detailed design

The design could be very simple. This is naive implementation based on public React API.

```jsx
import { useState, useEffect } from 'react';

export function createSharedState(defaultValue) {
  const listeners = new Set();
  let backupValue = defaultValue;

  return () => {
    const [value, setValue] = useState(backupValue);

    useEffect(() => {
      listeners.add(setValue);
      return () => listeners.delete(setValue);
    }, []);

    useEffect(() => {
      backupValue = value;
      listeners.forEach(listener => listener !== setValue && listener(value));
    }, [value]);

    return [value, setValue];
  };
}
```

For more details please check the [`react-create-shared-state`](https://github.com/mucsi96/react-create-shared-state) NPM package. 

# Drawbacks

As with any React hooks this works only in functional components. 

# Adoption strategy

This is not a breaking change just addition to API.
Migration from Context API to `createSharedState` can be done gradually in application.
The concept itself can be tested with [`react-create-shared-state`](https://github.com/mucsi96/react-create-shared-state) NPM package.

# How we teach this

React API documentation need to be extended with documentation for `createSharedState` function.
Also I recommend it's adoption in the tutorial and mentioning in Context API page.

# Unresolved questions

- How can we implement this with internal React API?