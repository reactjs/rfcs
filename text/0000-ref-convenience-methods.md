- Start Date: 2018-08-20
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Introduce a small API to the ref object returned by createRef() that will make handling references more convenient.


# Basic example

```jsx
import React, {Component} from 'react';
import {animateLinkPressed, animateLinkAppearance} from './animations'; // hypothetically

class RefExample extends Component {
	buttonOne = React.createRef();
	buttonTwo = React.createRef();
	
	constructor(props) {
		super(props);
		
		this.buttonOne.onReference(ref => {
			animateLinkAppearance(ref, 'green');
		});
		
		this.buttonTwo.onReference(ref => {
			animateLinkAppearance(ref, 'blue');
		});
	}
	
	onClick = (ref) => {
		animateLinkPressed(ref);
	}
	
	render() {
		return <div>
			<a ref={this.buttonOne} onClick={() => this.buttonOne.ifCurrent(onClick)}>Button One</a>
			<a ref={this.buttonTwo} onClick={() => this.buttonTwo.ifCurrent(onClick)}>Button Two</a>
		</div>
	}
}
```

# Motivation

After working with React for a few years, dealing with refs has become an area that we've noticed is a little more troublesome than most others.  We believe that it is a good area to target for ergonomic improvements that will alleviate some annoyances in React development.  We have targeted two areas: handling optional references and interacting with refs as they become available.

The ref object can be considered a sort of optional type as the current ref is optionally present.  React reference handling (particularly in the case of flow) often looks something like: `someRef.current && someRef.current.someMethod()` or `!someRef.current && doSomething()`.  This sort of code repetition is almost always a code smell.  Exposing optional type-like methods would make this pattern obsolete. 

It's occasionally necessary to interact with a reference after it's rendered, but without waiting on user input.  Some examples are scrolling to elements as they're rendered, starting playback on a video, or animating components.  There are some cases where handling refs in lifecycle methods is not easy because of conditional rendering or waiting for lifecycles in other parts of the tree to complete.  Introducing a callback method that is invoked when a ref is set would make this much easier.

A [search on Stack Overflow](https://stackoverflow.com/search?q=react+refs+undefined) shows dozens of results for people having issues with unset references.  We believe the suggested API could preempt most of them.

# Detailed design

Proposed new ref API:

```
	{
		current: ?React$Ref<R>, 
		
		// new additions 
		
		// optional handling
		
		// would be immediately invoked with the ref if the ref is valid
		ifCurrent(callback: <T>(React$Ref<R>) => T): T,
		// would be immediately invoked if the ref is invalid
		ifNull(callback: <T>() => T): T,
		
		// reference interaction callback
		// will be invoked whenever the ref changes or is set for the first time
		onReference(callback: (React$Ref<R>) => void): void,
	}

```

# Drawbacks

There is one primary drawback to this proposal: the increase in React API surface.  Any increase in API surface must be considered against its usefulness, and we believe these small additions handle enough common headaches to merit their consideration.  

As far as the cost of work, we don't believe the implementation cost will be that high: ifCurrent/ifNull can be static convenience methods attached to the ref object and won't require any deep changes to React, and onRef calls can be made in a setter on the ref object's current property, leaving the changes entirely encapsulated in a single file.  We can submit a patch if the RFC is accepted.

# Alternatives

The primary alternative is to leave React unchanged and make developers continue to handle references as they have been.


# Adoption strategy

It's not clear how far along React developers are with adopting createRef over callback refs, but as they adopt that API, they will also have this API available to them.  There are no backwards compatibility breaking changes suggested.  As the suggested API is primarily targeted for developer convenience, some minor marketing like a blog post to increase awareness may be warranted, but a concerted effort to drive adoption doesn't seem necessary.

# How we teach this

An API change this small should only need a few additions to the current documentation.  As the API becomes more popular, it may increase the complexity of teaching students ref handling, but that's a decision left to the teachers, as the new API is by no means mandatory or necessary for understanding the deeper concepts.  

# Unresolved questions

* Are the suggested API names acceptable?
