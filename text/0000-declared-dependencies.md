- Start Date: 2020-10-17
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Provide a hook that can be used to declare the relevant condition's a value's "equality" should change from the perspective of hooks using that value.

This RFC is about a potential common convention. I do not anticipate this RFC making a change to React. However I do wish to start a discussion about whether this pattern is a good idea, or what the best solution to these issues are.

# Basic example

```js
// Example hook representing any auth system someone may use, returns the current user's auth token and userId
const { token, userId } = useLoginState();

// Proposed hook
// - apiTokenRef will always contain the current value in `token`
// - however apiTokenRef will be referentially identical unless `userId` changes
const apiTokenRef = useDependencyDeclaration(token, [userId]);

// Crude example representing an API fetching effect
// Because apiTokenRef does not change as long as userId is the same, fetchSelf will not be re-executed if the token is just refreshed
// However because it does change when userId changes, fetchSelf will re-execute if the user signs in to a different account
const [user, setSelf] = useState(null);
const fetchSelf = useCallback(async () => {
  const response = await fetch('/me', {
    headers: {
      Authorization: `Bearer ${apiTokenRef.current}`
    }
  });
  setSelf(await response.json());
}, [apiTokenRef]);
useEffect(() => {
  fetchSelf();
}, [fetchSelf]);
```

# Motivation

Thanks to `react-hooks/exhaustive-deps` lint rule the dependencies of hooks are commonly managed automatcally and we are warned when we are missing a dependency that would cause an effect to not run when conditions change.

However occasionally there are circumstances where the referential equality checks used in deps do not match the conditions we actually want effects to be re-run on change of.

Currently when this issue arrises most developers workaround it by using `// eslint-disable-next-line react-hooks/exhaustive-deps` to disable the lint rule for one hook the author wishes to explicitly declare the change conditions for. This is unfortunate because if the hook in question has deps other than the one you want to control the conditions of, you will no longer be warned if a dep is missing. Additionally it should generally be considered bad to have to disable lint rules as as method of declaring intent.

Here is an example of that [from the axios-hooks](https://github.com/simoneb/axios-hooks/blob/9218707871750c472420bf1ae1a570969b96e3bf/src/index.js#L189-L199) library:

```js
config = React.useMemo(
  () => configToObject(config),
  // eslint-disable-next-line react-hooks/exhaustive-deps
  [JSON.stringify(config)]
)
```

However sometimes this issue goes beyond simply wanting to ensure that a dep does not change referential equality when the value is practically the same. Sometimes you have a value that can functionally change in a way you do want a callback to recieve the new value of, but do not want to result in any effects dependent on that callback to re-run because of.

A good example of this is API auth tokens. If an auth token is refreshed before it expires you want any callbacks that would call an API with that token to have access to the new token. But you do not want an effect (or Suspense fetching) to trigger another API call for the same data.

However if an auth token changes because the user has logged out or switched user accounts, then you do want effects that cause API calls to refetch with the new token because the content or the user's permissions to see that content has changed.

# Detailed design

The proposed implementation of this pattern would be a hook with this pattern:

```js
const ref = useDependencyDeclaration(value, deps);
```

- `value` is the value we wish to declare the change conditions for.
- `deps` is a standard deps array. However exhaustive-deps does not manage it because we are explicitly declaring the condition that referential equality should change.
- `ref` is a `Ref` that will act as our "value with defined deps"
- As long as deps are unchanged `ref` will always return the same ref, this will ensure that the deps of hooks using it do not change.
- The `current` value of `ref` will be kept up to date with the current value of `value`.
- When `deps` change a new ref is returned so the deps of hooks depending on this value will change. This also ensures that old callbacks dependent on the old value do not recieve the new value.

In theory the following code should fulfill these rules.

```js
function useDependencyDeclaration(value, deps) {
  const ref = useMemo(() => createRef(value), deps);

  useLayoutEffect(() => {
    ref.current = value;
  }, [value])

  return ref;
}
```

# Drawbacks

- Using refs results in a need to use `.current` more regularly than typical.
- Because refs need to be updated in effects, I do not know if there are any race conditions where an effect may run before the effect that updates the ref to the current value.

# Alternatives

1.
  The current status quo is still functional. Developers are able to disable lint rules when they need to declare explitict dependencies. And they can still manually define refs.

  However I think it is preferential to be able to declare deps without disabling linting rules. And it should be easier to manage these types of refs when needed.
2.
  Some of this could be done as part of react by making the `deps` array "smarter" allowing it to be passed special values that explicitly declare change conditions. But this implementation would probably be more complex than it should be.
3.
  Explicit dependencies could also be implemented by standardizing a series of special comments that declare explicit alterations to the dependency list. Which the lint rule could use to still run while avoiding altering deps arrays in ways that would change the intended result.

# Adoption strategy

If approved I would like to see this hook available in a common location for everyone building web apps and libraries use. Either in `react` or in another popular hooks library that some people may already have installed.

# How we teach this

It may be worth documenting this pattern in the react hooks documentation to educate people about it. Perhaps in an an "alternatives to breaking the rules of hooks" section which could document alternatives to disabling exhaustive deps and common mistakes in hooks. And the `react-hooks/exhaustive-deps` rule documentation should probably reference this.

I expect better names for terminology and the hook(s) may show up in documentation. But I recommend using terminology that implies you are declaring the attributes of a value which are relevant to that value changing.

# Unresolved questions

- The best naming for the hook is unknown
- The proposed hook is probably overcomplicated for simpler scenarios like how `axios-hooks` only needed memoize to ensure a value was still referentially identical if a config object had not actually changed settings. Should we define other hooks for these scenarios?
- Where should this hook be published. In react, a common hooks library. or a dedicated library with shared maintenance?
