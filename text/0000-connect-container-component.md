- Start Date: 2022-02-06
- RFC PR:
- React Issue:

# Summary

Add a special props named connectContainer to React.Component and extend createElement.

# Basic example

Giving that, you want child prop using hooks combined with redux state and define useOptions as a hook followed by example.

```tsx
export function useOptions = () => {
    const users = useSelector()
    return useMemo(() => {
        return users.map((user) => {
            return {
                id: user.id,
                label: user.name
            }
        })
    }, [users])
}
```

```tsx
import { useOptions } from "./useOptions";

const Example = (props) => (
  <Parent>
    <Component
      connect={() => ({
        options: useOptions(),
      })}
    />
  </Parent>
);
```

nearly equals to

```tsx
import { useOptions } from "./useOptions";

const Example = (props) => {
  const options = useOptions();
  return (
    <Parent>
      <Component options={options} />
    </Parent>
  );
};
```

but, they are different about execution scopes. The formar is in child scope and the latter is in parent scope.
In addition, connectContainer prop must return partial props of the component.

# Motivation

Component often needs conditional renderning but hooks must be written before their even if they don't depend on the condition for [idempotent calling rule of hooks](https://reactjs.org/docs/hooks-rules.html).

OK:

```tsx
const Example = (props) => {
  const options = useOptions();
  const { initialized, data } = useFetchData();
  if (!initialized) return null;
  return <Component {...data} options={options} />;
};
```

Bad:

```tsx
const Example = (props) => {
  const { initialized, data } = useFetchData();
  if (!initialized) return null;
  const options = useOptions();
  return <Component {...data} options={options} />;
};
```

or

```tsx
const Example = (props) => {
  const { initialized, data } = useFetchData();
  if (!initialized) return null;
  return <Component {...data} options={useOptions()} />;
};
```

This is not problem when component is small, but big one is tough to read.

```tsx
const Example = (props) => {
    const options = useOptions()
    const [optionValue, setOptionValue] = useState()
    const {initialized, data} = useFetchData()
    const someValue = ''
    if (!initialized) return null
    return (
        <Component>
            <Child>
              <AnnoyedField
                value={someValue}
                onChange={someChange}
                class='test'
                otherProps
              />
              <AnnoyedField
                value={someValue}
                onChange={someChange}
                class='test'
                otherProps
              />
              <AnnoyedField
                value={someValue}
                onChange={someChange}
                class='test'
                otherProps
              />
              <AnnoyedField
                value={someValue}
                onChange={someChange}
                class='test'
                otherProps
              />
              <AnnoyedField
                value={someValue}
                onChange={someChange}
                class='test'
                otherProps
              />
              <AnnoyedField
                value={someValue}
                onChange={someChange}
                class='test'
                otherProps
              />
              <AnnoyedField
                value={someValue}
                onChange={someChange}
                class='test'
                otherProps
              />
              <Select
                value={optionValue}
                onChange={setOptionValue}
                options={options}
                otherProps
              />
              <AnnoyedField
                value={someValue}
                onChange={someChange}
                class='test'
                otherProps
              />
              <AnnoyedField
                value={someValue}
                onChange={someChange}
                class='test'
                otherProps
              />
            <Child/>
        </Component>
    ) {...data} options={useOptions()} />
}
```

As the farther place it's used from definition, it's more tough to remember the variable name and we are forced to use editor's trick like code jump, bookmark, splited view, etc. Or scroll and switch page many times.

The RFC way can envelop independent hooks against others in narrower scope.

```tsx
const Example = (props) => {
    // ... Omitted
    const {initialized, data} = useFetchData()
    if (!initialized) return null
    return (
        <Component>
        {/* ...Omitted */}
            <Select
                connectContainer={() => {
                    const [optionValue, setOptionValue] = useState()
                    const options = useOptions()
                    return {
                        options,
                        value: optionValue,
                        onChange: setOptionValue
                    }
                }}
                otherProps
            />
        {/* ...Omitted */}
        </Component>
    )
```

And this is more useful to move the component to other place compared to the past. This example may be meaningless because state created by useState scope is very limited. It's for sake of simplicity. If you use wide scope state and handler like redux or recoil or widely-context, you'll find this feature powerful. Now we can use hooks in context instead of extracting components per container or context's consumer which tends to become a source of trouble about optimizing rendering and readability. For example, we can write context to the extent of root component scope without separationg files.

```tsx
const Context = createContext ()
const Example = (props) => {
    // ... Omitted
    const {initialized, data} = useFetchData()
    if (!initialized) return null
    return (
        <Context.Provider value={{value: 'any'}}>
          {/* ...Assuming a vast of <AnotherFieldNoDependsOnContext /> */}
          <Field
              connectContainer={() => {
                  const {value} = useContext(Context)
                  return {
                      value,
                  }
              }}
          />
          {/* ...Assuming a vast of <AnotherFieldNoDependsOnContext /> */}
        </Context.Provider>
    )
```

This can limit context scope and are easier to know and extract loosely-coupled component compared to followed by the context example, we could just only use provider in the past, and it's easy to miss how many we set curly braces for deep nests.

```tsx
const Context = createContext ()
const Example = (props) => {
    // ... Omitted
    const {initialized, data} = useFetchData()
    if (!initialized) return null
    return (
        <Context.Provider value={{value: 'any'}}>
          <Context.Consumer>
            {
              ({value}) => {
                return (
                  <>
                    {/* ...Assuming a vast of <AnotherFieldNoDependsOnContext /> */}
                    <Field
                        connectContainer={() => {
                            const {value} = useContext(Context)
                            return {
                                value,
                            }
                        }}
                    />
                    {/* ...Assuming a vast of <AnotherFieldNoDependsOnContext /> */}
                  </>
                )
              }
            }
          </Context.Consumer>
        </Context.Provider>
    )
```

There is simular problem about readability even if you use consumer per components.

# Detailed design

The connectContainer is called in intermediate layer from parent and child.
React render function create component only call hooks and merge props passed by innerProps.

```tsx
function InterMediate(props) {
    const propsFromInnerHooks = connectContainer && connectContainer()
    return <Child {...props} {...propsFromInnerHooks}>
}
```

In api level, we may be able to optimize performance.

```tsx
React.createElement(Hello, { connectContainer: someHandleHookFunction }, null);

function createElement(Component, props, ...children) {
  if (props.connectContainer) {
    props = merge(props, props.connectContainer());
  }
  // continue to original function process
}
```

# Drawbacks

- possible to be worse rendering performance by intermediate hooks
- parent scope variables included in connectContainer can perfome unexpected side effects.
- type inference is more complicated, it needs returned type of connectContainer and exclude them from original props if it injected

# Alternatives

No idea.

# Adoption strategy

- This does not have any breaking changes. It can ignore if there is not an connectContainer prop, unless users name a prop connectContainer.
- React typing library like flow and typescript should change component type so that infer props type when exists the connectContainer prop.

# How we teach this

The terminology is connectContainer.

We may need only adding connectContainer usage the React Hooks entry of document.

And introduction as new feature is enough.

# Unresolved questions

- How to imprement connectContainer during rendering.
- I may not consider enough how extent connectContainer side effects by connectContainer of decendants from parent have bad effects.
