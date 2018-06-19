- Start Date: 2018-06-19
- RFC PR:
- React Issue:

# Summary

This RFC seeks to add the function `createSingleton` to the React namespace. This function returns a unique React component which could be used anywhere but its children would only get rendered a single time in the DOM for every location the returned Singleton is used. It is very useful for declaring SVG filters/gradients and can be combined with other technologies including [Portals](https://reactjs.org/docs/portals.html) and [Fragments](https://reactjs.org/docs/fragments.html).

# Basic example

In this example, a singleton is created within `FancyEffect`, which is used throughout the application, however, only a single instance of the SVG is rendered to the DOM.

```jsx
const FilterSingleton = React.createSingleton();

const FancyEffect = () =>
{
	return (
		<div>
			<FilterSingleton>
				<svg className='svg-defs' xmlns='http://www.w3.org/2000/svg'>
					<defs><filter id='can-be-used-anywhere'></filter></defs>
				</svg>
			</FilterSingleton>

			<SomeComponent ... />
			<SomeComponent ... />
			<SomeComponent ... />
		</div>
	);
};

// Used all throughout your application but SVG is only rendered at most once on the DOM
<FancyEffect />
<FancyEffect />
<FancyEffect />
```

# Motivation

The basic goal here is to develop an easier way of creating self-contained components that require rendering something to the DOM a single time. For example, consider a [Ripple Effect](https://www.webcomponents.org/element/PolymerElements/paper-ripple) which could get applied to anything including a `div` and a `button`. This effect might make use of an SVG filter to make it [gooey](https://css-tricks.com/gooey-effect). Using `createSingleton` here makes it easy to conditionally render the SVG filter once in the DOM, within the same logic flow as the rest of the Ripple and have it be be shared among all other Ripples.

# Detailed design

The `createSingleton` function would return a unique component which would keep track of the places it's being rendered. The component would render its children if it's the first component in the list, otherwise it would render null. If the component rendering children gets unmounted, it's removed from the list and the next component in the list does a [`forceUpdate`](https://reactjs.org/docs/react-component.html#forceupdate). Below is a sample implementation:

```jsx
React.createSingleton = function()
{
	const instances = [ ];

	return class Singleton extends PureComponent
	{
		constructor (props) {
			super (props);
			instances.push (this);
		}

		componentWillUnmount() {
			const index = instances.indexOf (this);

			if (index > -1) {
				instances.splice (index, 1);
			}

			if (instances[0]) {
				instances[0].forceUpdate();
			}
		}

		render() {
			return instances[0] === this ? this.props.children : null;
		}
	};
};
```

# Drawbacks

There's no way to explicitly control where in the DOM the code gets rendered. In the case of SVG's or portals it doesn't matter but it might matter if the code is dependent on being rendered in a specific location. Currently, the first instance of the singleton is responsible for rendering the code but it might be necessary to add some sort of preference property to manipulate the location of the rendered code. But doing this would add more complexity to this feature and might not really be needed.

# Alternatives

Other designs included creating a single component with a property identifying its uniqueness. The problem with doing this is potential for name collisions and React in general has started using instances more in general such as with [`createRef`](https://reactjs.org/docs/refs-and-the-dom.html#creating-refs). Below is an example of how singleton's would work if they were a single component.

```jsx
<Singleton key='mySingleton'>
	<svg className='svg-defs' xmlns='http://www.w3.org/2000/svg'>
		<defs><filter id='can-be-used-anywhere'></filter></defs>
	</svg>
</Singleton>
```

# Adoption strategy

This is an entirely new feature, so backward compatibility is not a concern. It's purely an opt-in feature for some specific use cases.

# How we teach this

Adding it as a new page to the React documentation would be sufficient. A blog post introducing this feature would also make sense.

# Unresolved questions

Is using `forceUpdate` in the proposed implementation, or the implementation in general, work correctly? Or are there edge cases with using this technique?
