- Start Date: 2018-02-21
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

This is a WIP RFC to discuss how we can develop the React ecosystem with the upcoming ECMA262 feature "Decorators".

[Decorators](https://github.com/tc39/proposal-decorators) is an ECMA262 proposal at stage 2 and yet to be standardised.

**Please note that decorators proposal have been through breaking changes. Please read the current version if you were unaware: [Link to Decorators Proposal](https://github.com/tc39/proposal-decorators)**

While it's not yet a standard, how they could be used in React ecosystem should be discussed for benefit of the React community and the standardisation process.

Just like React uses JSX at its core, it could also make use of another language future, decorators to enable better development experiences.

# Examples

## React Use: Component Flags

Instead of extending a separate class or nesting components for flagging a component, decorators can be used:
```javascript
// Before:
export default class MyComponent extends React.PureComponent {
  return (
    <AsyncMode>
      {/* ... */}
    </AsyncMode>
  )
}

// After:
@React.pure
@React.async
export default class MyComponent extends React.Component {}
```

## React Use: Refs

A field decorator can be implemented to allow easier use of Refs:
```javascript
class MyComponent extends Component {
  @React.ref
  myTextInput;

  myMethod = () => {
    foo(this.myTextInput.value);
  }

  render() {
    return (
      <TextInput ref={this.refs.myTextInput} />
    )
  }
}
```

## Community Use: HOCs

Many libraries like Redux and Relay provide additional Props to Components. They do this by wrapping the Component in another Component. Class decorators can be used to simplify consumption.

```javascript
// Before:
import { withRouter, connect } from './hocs';

class MyComponent extends Component { /* ... */ }

const MyComponentWithRouter = withRouter(MyComponent);

export default connect(/* ... */)(MyComponentWithRouter);

// After
import { withRouter, connect } from './hocs';

@withRouter
@connect(/* ... */)
export default class MyComponent extends Component { /* ... */ }
```

## Community Use: Providing Mock Data

A decorator can be implemented for attaching mock data to a Component to help out-of-context rendering:

```javascript
type Props = {
  showDetails: string, // From Redux
  user: User, // From Relay
}

@MyLib.mock({
  showDetails: false,
  user: MyLib.fakeUser,
})
class MyComponent { /* ... */ }
```

Out-of-context rendering enabled by using such a decorator can allow design tools (like Facebook Origami and Adobe XD) or IDEs (like Atom and VSCode) to work on individual files.

# END OF DOCUMENT, THE REST WILL BE FILLED LATER

# Motivation

Why are we doing this? What use cases does it support? What is the expected
outcome?

Please focus on explaining the motivation so that if this RFC is not accepted,
the motivation could be used to develop alternative solutions. In other words,
enumerate the constraints you are trying to solve without coupling them too
closely to the solution you have in mind.

# Detailed design

This is the bulk of the RFC. Explain the design in enough detail for somebody
familiar with React to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

# Drawbacks

Why should we *not* do this? Please consider:

- implementation cost, both in term of code size and complexity
- whether the proposed feature can be implemented in user space
- the impact on teaching people React
- integration of this feature with other existing and planned features
- cost of migrating existing React applications (is it a breaking change?)

There are tradeoffs to choosing any path. Attempt to identify them here.

# Alternatives

What other designs have been considered? What is the impact of not doing this?

# Adoption strategy

If we implement this proposal, how will existing React developers adopt it? Is
this a breaking change? Can we write a codemod? Should we coordinate with
other projects or libraries?

# How we teach this

What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing React patterns?

Would the acceptance of this proposal mean the React documentation must be
re-organized or altered? Does it change how React is taught to new developers
at any level?

How should this feature be taught to existing React developers?

# Unresolved questions

Optional, but suggested for first drafts. What parts of the design are still
TBD?
