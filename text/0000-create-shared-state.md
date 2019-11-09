- Start Date: 2019-11-08
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

I would like to take a step forward with idea behind hooks by replacing the Context API with custom hooks.
I suggest to `useState` hook with shared state across the application.

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

`createSharedState` results with a cleaner codebase with easily reusable shared state. It solves the following concerns with current Context API:  
- Developer need to think where to place the `Provider` component in the tree structure.
- There is a possibility for placing it in wrong place which will result in errors or performance issues.
- Sometimes the DOM structure need to be restructured because of it.
- Sharing context between two unrelated deeply nested components may require placing `Provider` in the root component which will result in unnecessary rerenders of the whole application.
- Often many providers are nested which makes the code hard to read.
- Testing a component which depends on context is harder due to necessary wrapping with required providers.

In React community many projects are already created with very similar idea:
- https://github.com/pie6k/hooksy
- https://github.com/kelp404/react-hooks-shared-state
- https://github.com/philippguertler/react-hook-shared-state
- https://github.com/dimapaloskin/react-shared-hooks
- https://github.com/atlassian/react-sweet-state
- https://github.com/paul-phan/global-state-hook
- https://github.com/donavon/use-persisted-state


# Detailed design

The design could be very simple. This is naive implementation based on public React API.

```jsx
import { useState, useEffect } from 'react';

export function createSharedState(defaultValue) {
  const listeners = [];
  let backupValue = defaultValue;

  return () => {
    const [value, setValue] = useState(backupValue);

    useEffect(() => {
      listeners.push(setValue);

      return () => {
        const index = listeners.findIndex(listener => listener === setValue);
        listeners.splice(index, 1);
      };
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

- Is this function covers all use cases of Context API?
- Can we easily migrate from Context API in every case? (not considering class based components)
- How can we implement this with internal React API?