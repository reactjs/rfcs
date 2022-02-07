- Start Date: 2022-0207
- RFC PR:
- React Issue:

# Summary

A new hook create component state and it can be passed decendants components via hooks.

# Basic example

Type inference improvement and linter violation hooks rule changes are needed by generic instead of const inference, but it works.

New hook api:

```tsx
function useDynamicStateFactory<T extends object>(
  initialState: Partial<T> = {}
) {
  const [state, setState] = useState<Partial<T>>(initialState);

  return [
    state,
    <K extends keyof T = Extract<keyof T, string>>(
      stateName: K,
      initialValue?: T[K]
    ) => {
      // InitalValue is priorer than first initialization.
      useEffect(() => {
        initialValue !== undefined &&
          setState((state) => {
            return {
              ...state,
              [stateName]: initialValue,
            };
          });
      }, [initialValue]);
      return [
        state[stateName],
        (value: T[K] | ((arg: T[K]) => T[K])) => {
          typeof value !== "function"
            ? setState({ ...state, [stateName]: value })
            : setState((state) => ({
                ...state,
                [stateName]: (value as any)(state[stateName]),
              }));
        },
      ] as const;
    },
  ] as const;
}
```

```tsx
import "./styles.css";
import React, { useEffect, useState } from "react";

function useDynamicStateFactory<T extends object>(
  initialState: Partial<T> = {}
) {
  const [state, setState] = useState<Partial<T>>(initialState);

  return [
    state,
    <K extends keyof T = Extract<keyof T, string>>(
      stateName: K,
      initialValue?: T[K]
    ) => {
      // InitalValue is priorer than first initialization.
      useEffect(() => {
        initialValue !== undefined &&
          setState((state) => {
            return {
              ...state,
              [stateName]: initialValue,
            };
          });
      }, [initialValue]);
      return [
        state[stateName],
        (value: T[K] | ((arg: T[K]) => T[K])) => {
          typeof value !== "function"
            ? setState({ ...state, [stateName]: value })
            : setState((state) => ({
                ...state,
                [stateName]: (value as any)(state[stateName]),
              }));
        },
      ] as const;
    },
  ] as const;
}

function Child(props: any) {
  return (
    <div>
      <input type={props.type} onChange={props.onChange} value={props.value} />
    </div>
  );
}

function Text(props: any) {
  return <div>{props.value}</div>;
}

function withInnerHooks(Child: React.ElementType) {
  return (props: any) => {
    const ex = props.innerHooks?.();
    return <Child {...props} {...ex} />;
  };
}

const NumberField = withInnerHooks(Child);
const StringField = withInnerHooks(Child);
const Timer = withInnerHooks(Text);

export default function App() {
  const [state, dynamicStateFactory] = useDynamicStateFactory<{
    num: number;
    str: string;
    timer: number;
  }>({
    num: 1,
    str: "foo",
    timer: 0,
  });

  return (
    <div className="App">
      <form
        onSubmit={(e) => {
          e.preventDefault();
          // Psuedo logic submit
          const submit = console.log;
          submit(state);
        }}
      >
        <NumberField
          type="number"
          innerHooks={() => {
            const [value, setValue] = dynamicStateFactory("num", 0);
            return {
              value,
              onChange: (e: any) => setValue(e.target.value),
            };
          }}
        />
        <StringField
          type="string"
          innerHooks={() => {
            const [value, setValue] = dynamicStateFactory("str");
            return {
              value,
              onChange: (e: any) => setValue(e.target.value),
            };
          }}
        />
        <input type="submit" />
      </form>
      <Timer
        innerHooks={() => {
          const [value, setValue] = dynamicStateFactory("timer");
          // This is warned by linter but it can be used.
          useEffect(() => {
            const i = setInterval(() => {
              setValue((state: number) => state + 1);
            }, 1000);
            return () => {
              clearInterval(i);
            };
          }, []);
          return {
            value,
          };
        }}
      />
    </div>
  );
}
```

# Motivation

Hooks can't be called if calling count changes during rendering in same scope.
For this, we must write hooks before conditional rendering description like followed by an example.

```tsx
const Example = (props) => {
  const { initialized, data } = useFetchData();
  // OK:
  const [options, _] = useState([...props.data]);
  if (!initialized) return null;
  // BAD: const [options, _] = useState([...props.data])
  return <Component {...data} options={options} />;
};
```

For this situation, we must write state or handlers outside of root component even if it's used by only one child component.
This often cause to make code readablity worse. So my goal is able to locate independent state from other siblings in the child's scope. Combined with [Inner Hooks Proposal](https://github.com/reactjs/rfcs/pull/210/files), generating partial state and the hander as hooks and inject child directly are powerful features of component potability.

Our proposal can do this, because generating state are executed in intermidate or child scope and keep idempotent like followed by an example.

```tsx
const Example = (props) => {
    const {initialized} = useIsLoaded()
    const [state, dynamicStateFactory] = useDynamicStateFactory<{
      num: number;
      str: string;
      timer: number;
    }>({
      num: 1,
      str: "foo",
      timer: 0
    });
  if (!initialized) return null;
  return (
    <StringField
      type="string"
      innerHooks={() => {
        const [value, setValue] = dynamicStateFactory("str");
        return {
          value,
          onChange: (e: any) => setValue(e.target.value)
        };
      }}
    />
  )
)
```

This reason is that dynamicStateFactory is called in intermediate layer of component has generate prop.

# Detailed design

In above example, you can see useDynamicStateFactory returns [state, dynamicStateFactory]. The useDynamicStateFactory can recieve initial whole state and the first state is same unless you initiailize by dynamicStateFactory.

```tsx
const [state, dynamicStateFactory] = useDynamicStateFactory({
  num: 1,
  str: "foo",
  timer: 0,
});
```

The dynamicStateFactory can specify state property name in whole state and provide it if exists or generate it with initialValue specified, then modifier function

The dynamicStateFactory can add propery as named by first argument of dynamicStateFactory and initialValue by second argument which overrides them of useDynamicStateFactory.

```tsx
const [value, setValue] = dynamicStateFactory("str", "initialized");
```

The reason why is that keep loosely-coupled to move easily and more intuitive after calling is prior than before doing.
Thus, you can define state and handler in seperated components per them, see the first example again.

```tsx
import "./styles.css";
import React, { useEffect, useState } from "react";

function useDynamicStateFactory<T extends object>(
  initialState: Partial<T> = {}
) {
  const [state, setState] = useState<Partial<T>>(initialState);

  return [
    state,
    <K extends keyof T = Extract<keyof T, string>>(
      stateName: K,
      initialValue?: T[K]
    ) => {
      // InitalValue is priorer than first initialization.
      useEffect(() => {
        initialValue !== undefined &&
          setState((state) => {
            return {
              ...state,
              [stateName]: initialValue,
            };
          });
      }, [initialValue]);
      return [
        state[stateName],
        (value: T[K] | ((arg: T[K]) => T[K])) => {
          typeof value !== "function"
            ? setState({ ...state, [stateName]: value })
            : setState((state) => ({
                ...state,
                [stateName]: (value as any)(state[stateName]),
              }));
        },
      ] as const;
    },
  ] as const;
}

function Child(props: any) {
  return (
    <div>
      <input type={props.type} onChange={props.onChange} value={props.value} />
    </div>
  );
}

function Text(props: any) {
  return <div>{props.value}</div>;
}

function withInnerHooks(Child: React.ElementType) {
  return (props: any) => {
    const ex = props.innerHooks?.();
    return <Child {...props} {...ex} />;
  };
}

const NumberField = withInnerHooks(Child);
const StringField = withInnerHooks(Child);
const Timer = withInnerHooks(Text);

export default function App() {
  const [state, dynamicStateFactory] = useDynamicStateFactory<{
    num: number;
    str: string;
    timer: number;
  }>({
    num: 1,
    str: "foo",
    timer: 0,
  });

  return (
    <div className="App">
      <form
        onSubmit={(e) => {
          e.preventDefault();
          // Psuedo logic submit
          const submit = console.log;
          submit(state);
        }}
      >
        <NumberField
          type="number"
          innerHooks={() => {
            const [value, setValue] = dynamicStateFactory("num", 0);
            return {
              value,
              onChange: (e: any) => setValue(e.target.value),
            };
          }}
        />
        <StringField
          type="string"
          innerHooks={() => {
            const [value, setValue] = dynamicStateFactory("str");
            return {
              value,
              onChange: (e: any) => setValue(e.target.value),
            };
          }}
        />
        <input type="submit" />
      </form>
      <Timer
        innerHooks={() => {
          const [value, setValue] = dynamicStateFactory("timer");
          // This is warned by linter but it can be used.
          useEffect(() => {
            const i = setInterval(() => {
              setValue((state: number) => state + 1);
            }, 1000);
            return () => {
              clearInterval(i);
            };
          }, []);
          return {
            value,
          };
        }}
      />
    </div>
  );
}
```

In addition, you can notice type error if you typing in useDynamicStateFactory when adding new named state. This is useful to prevent bug by a little mistakes like typo. If you write components like avobe examples, they can be used everywhere by coping and paste only in javascript (All you may need to do is some type changes of useDynamicStateFactory even if in typing environment).

# Drawbacks

- These states can be created by only scope can define hooks in decendants components, then if you use this api in roughly many props of a component. You sholud easily violate calling hooks idempotent rule. So, you should use in the regulation that this api only in [inner hooks prop in this proposal](https://github.com/reactjs/rfcs/pull/210/files).
- useDynamicStateFactory using hooks(useEffect) in closure in custom hooks, thus this is catched as violation in the current linter rule. It can be avoidable by using useRef though it is messy and useless binding variables.

# Alternatives

No Idea.

# Adoption strategy

Some lint rules of violation rules eslint-react needs changes for innerHooks and useDynamicStateFactory (though it may be enough take useDynamicStateFactory as an only exception).

# How we teach this

As a new feature of react in document.

# Unresolved questions

- How hard to remove warning by hooks calling regulation for only innerHooks and using hooks in returned closure.
- What are there problems that using hooks in closure?
- My proposal generate funation name dynamicStateFactory it can have any name, but it is hooks so the variable name should rename use~ in this document? The useDynamicStateFactory in my definition doesn't really means that `dynamic` because useStateAndHandlerFactory hooks the factory created are necessary to be idempotent about calling count. So what is appropriately called?
