- Start Date: 2018-07-12
- RFC PR:
- React Issue: 

# Summary

Currently React throws warnings when `null` is used as a "value" on a controlled
component. This RFC brings [this issue](https://github.com/facebook/react/issues/11417) 
officially into the RFC process.

Note: Much of this RFC is a copy of key quotes from the original issue.   

# Basic example

@IndifferentDisdain :
> Currently, if you create an input like `<input value={null} onChange={this.handleChange} />`,
> the null value is a flag for React to treat this as an uncontrolled input, and a console
> warning is generated. However, this is often a valid condition. [...]
> Address Line 2 is often optional. As such, passing null as value to this controlled component
> is a very reasonable thing to do.

> One can do a workaround, i.e. `<input value={foo || ''} onChange={this.handleChange} />`, but
> this is an error-prone approach and quite awkward.

# Motivation

See https://github.com/facebook/react/issues/5013#issuecomment-340898727

@gaearon :
> There's no problem with null as a value. The problem is React treats this as a flag to make an
> input uncontrolled. That turned out to be confusing.
  
> So we added a warning to discourage existing users from passing null. This lets us safely
> change the behavior for future version to treat null as an empty string. We haven't done this
> yet but that's the plan, if I understand it right.

@paulirwin :
> ...it does make more sense with input type number where using empty string to represent a 
> missing number value feels counter-intuitive: `<input type="number" value={typeof foo === "number" ? foo : ""} />`
>
> (Aside: it would also be nice to be able to bind Date objects to the value of
> `<input type="date" />`, and I only bring it up because that would make this case even stronger.)

@ryannerd :

In SQL `null` is a fundamental concept. You never store an empty string in something like an
`Address2` column; this would negatively impact storage space, and performance. Another example
of `null` being a valid "value" is that averages, sums and other aggretate calculations exclude
nulls to perform valid calcuations (`0 !== null`). This is important for React since persistent
storage for many React apps is SQL based and thus would benefit from `null` as an allowed value. 

@nickgcattaneo :
> I face this issue often when building controlled input's who rely on a record from state slice
> (such as an Immutable.Record via redux).
> ... I usually just set in my record the default value as '', but it would be great to use null
> since I can tell (without a pristine flag) if the field is pristine or simply been edited based
> on this concept.

@rdsedmundo :
> This problem happens to me very often when I'm using redux-form ...
> I have to change initial values to either undefined (making them uncontrolled them) or to
> empty string even when situations that they aren't strings. It works because at the beginning,
> on an input everything is a string, but it's counterintuitive.

# Detailed design

The majority of the design is internal to React itself, and was under discussion by the core
developers. The two design choices are:

1. Always treat `value={null}` as empty string for controlled components.
2. @IndifferentDisdain :
   > ... use `value` for controlled inputs and `defaultValue` for uncontrolled

# Drawbacks

- Possible breaking change for existing projects(?)
- @sophiebits :
> I don't think it's obvious which is better: I could imagine a case where you expect 
> `value={null} defaultValue="foo"` renders an input with an (uncontrolled) value of foo.
> Treating null as controlled empty string would be surprising then. Also, all of our
> host/DOM components treat null and undefined identically right now, I believe. 
> Don't know if we want to deviate from that.  

# Alternatives

@ernestopye
> To avoid breaking existing projects, would it be possible perhaps to have an attribute that
> allowed you to specify that the input is controlled, regardless of the value?
>
> `<input forceControlled type="text" value={someNullValue} onChange={this.myChangeHandler} />`

# Adoption strategy

This issue was already under discussion by the React core developers, and slated for 
possible inclusion in version 16, but may have fallen off the radar.
  
# How we teach this

When implemented this feature change would be documented on the [React Blog](https://reactjs.org/blog).

# Unresolved questions

If the design choice is made where `value={null}` is always treated as string:
* What about the situation where the controlled component is expecting a number or date?
  Always treating `value={null}` as a string could be problematic.

If the design choice is `value` for controlled and `defaultValue` for uncontrolled: 
* What should be done in the case `value=undefined`? (throw a warning or an error, do nothing)

For the alternitive design choice of `forceControlled` attribute:
* What is the impact if this choice is made?