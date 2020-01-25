- Start Date: 2020-01-18
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary
Adding compilation solutions to improve developer experience. The RFC considers the following:
- Adding new syntactic features
- As an alternative, leveraging Babble Macros.

# Motivation
React is well known (and rightly so) for enabling declarative UI programming. However, certain aspects of programming in modern React require technical, verbose, and in certain cases trivial code. Such is the case when trying to update a nested state value immutably, or when passing dependencies to hooks.

JSX shows how we can significantly improve developer experience in React. 
JSX has introduced an elegant syntax-compilation solution. 
```jsx
<element prop="value">child</element>
```
Compiles to: 
```js
React.createElement("element", {prop:"value", children:"child"})
```
This solution allows us to enjoy both worlds - stay in JS, yet use dedicated syntax which makes our code easier to read and write. This RFC is about extending this (syntax-compilation) pattern to the basic hooks, and state management in React to significantly improve developer experience. User will be able to opt-in to easy to use compilation solutions, by using dedicated syntax. 

Hooks are ideal candidates: 
1. Hooks come with non trivial constraints (Returned value depends on execution order, so they must be at the top level). With the use of a dedicated syntax, Hooks can be grasped and treated as different from a regular function call.  
2. Optimizing is more difficult - Optimizing for performance requires knowledge and attention to implementation details, including: React.memo, useCallback, useMemo, and passing dependencies. Optimizing requires more code, and is somewhat more [bug prone](https://stackoverflow.com/questions/53070970/infinite-loop-in-useeffect).

Another ideal candidate is state management, which comes with it's own necessary constraints. Most notably is that updating state should be done immutably. While this is essential in React, it could also be a burden on the developer, and is bug prone. This could also be elegantly solved with compilation. 

From a different perspective, React relies on immutability and memoization for performance, but both require technical verbose code. While immutability and memoization are not native to JS, we can provide React developers with an environment in which memoization and immutability feel native.

# Basic example
The current example and syntax might be controversial but demonstrate the general proposal, and indicate a few possible directions. 

Below is a hypothetical "Complete and Share" app (a to do app, with two features - There are subtasks, and there is a social aspect - The user can view stats regarding the number of tasks his contacts have completed, and inform them accordingly.)

```jsx
const CompleteAndShare = ({contactsCompletedStats, contacts}) => {
	const [tasks, setTasks] = useState(initialTasks)


	const toggleCompleted = useCallback(
		(id) => setTasks( 
			tasks => tasks.map( 
				task => (task.id!=id)? task : {...task, completed: !task.completed}
			)
		), 
		[]
	)

	const editSubTask = useCallback(
		(taskId, subTaskId, newText) => setTasks( 
			tasks => tasks.map( 
				(task, i) => (taskId!=i)? task : 
					{...task, subTasks: task.subTasks.map ( (subTask, j) =>
						(subTask.id ===j)? subTask : {...subTask, newText}
					)}
			)
		), 
		[]
	)

	const completedTasks = useMemo(
		()=>tasks.reduce(
			(accum, task) => accum+task.completed,
			0
		), 
		[tasks]
	)

	const shareCompletedTasks = useCallback(
		(contactId) => API.shareCompleted(contactId, completedTasks), 
		[completedTasks]
	)

	return (
		<>
			<h1>Complete and share</h1>
			<ControlPanel shareCompletedTasks={shareCompletedTasks} contacts={contacts} />

			<ContactStats contactsCompletedTasks={contactsCompletedTasks} />
			<SelfStats completedTasks={completedTasks} />
			
			<TaskList tasks={tasks} toggleCompleted={toggleCompleted} editSubTask={editSubTask} />
		</>
	)
}

export default React.memo(CompleteAndShare)
```

Below is the same functionality, relying on syntax-compilation to solve the challenges mentioned above. We shall use the following syntax for: 
`S:` - Declaration of a state entity 
`M:` - Declaration of a memoized entity

```jsx
M: CompleteAndShare = ({contactsCompletedStats, contacts}) => {
	s: tasks = initialTasks

	M: toggleCompleted = (id) => tasks[id].completed = !tasks[id].completed
	
	M: editSubTask = (taskId, subTaskId, newText) => tasks[taskId][subTaskId].text = newText

	M: completedTasks = tasks.reduce(
			(accum, task) => accum+task.completed,
			0
		)


	M: shareCompletedTasks = (contactId) => API.shareCompleted(contactId, completedTasks)

	return (
		<>
			<h1>Complete and share</h1>
			<ControlPanel shareCompletedTasks={shareCompletedTasks} contacts={contacts} />

			<ContactStats contactsCompletedTasks={contactsCompletedTasks} />
			<SelfStats completedTasks={completedTasks} />
			
			<TaskList tasks={tasks} toggleCompleted={toggleCompleted} editSubTask={editSubTask} />
		</>
	)
}

export default CompleteAndShare
```

It is significantly shorter, but more important, it includes less implementation details, and is easier to write, read and understand. 

# Detailed design

## Memoisation
In the example below:
```jsx
M: shareCompletedTasks = (contactId) => API.shareCompleted(contactId, completedTasks)
```
the compiler needs to memoize `shareCompletedTasks`. `shareCompletedTasks` is a function and has one dependency (`completedTasks`). It can therefore convert to: 

```jsx
const shareCompletedTasks = useCallback(
	(contactId) => API.shareCompleted(contactId, completedTasks), 
	[completedTasks]
)
```

However, using compilation enables new patterns:
1. The function that is passed to useCallback is recreated each time, even if an older version is used. Recreation could be avoided if the compiler moves the function outside the component body. 

```jsx
const memoizedCompletedTasks = generalMemoisingFunction(
	completedTasks => contactId => API.shareCompleted(contactId, completedTasks)
)

const TasksApp = {
	...
	const shareCompletedTasks = memoizedCompletedTasks(completedTasks)
	...
}
```
2. Currently the returned value depends on execution order. Consequently, inline callbacks can not be memoized, as the element might be conditionally rendered. The pattern suggested in section 1 does not rely on execution order so can also work for inline callbacks.
```jsx
	<ControlPanel 
		shareCompletedTasks={
			M: (contactId) => API.shareCompleted(contactId, completedTasks)
		} 
		contacts={contacts} 
	/>
```

In our new CompleteAndShare example all memoized values share the same syntax. Based on the memoized object, the compiler can wrap the code in React.memo, useCallback or useMemo. This syntax-compilation pattern hides memoization implementation details, and frees the developer from worrying about it. 

## State
In the example below: 
```jsx
S: tasks = initialTasks

M: editSubTask = (taskId, subTaskId, newText) => tasks[taskId][subTaskId].text = newText
```
tasks is defined as a state entity. The compiler will convert the declaration to: 
```jsx
const [tasks, setTasks] = useState(initialTasks)
```

The compiler recognizes that a state entity is being mutated/updated in editSubTask. It should first convert to: 
```jsx
M: editSubTask = (taskId, subTaskId, newText) => setTasks( tasks => 
	tasks[taskId][subTaskId].text = newText
)
```
All that is left is to convert mutating code to an immutable update. Immer has shown it to be possible in modern JS. One possible implementation is shown below: 
```jsx
M: editSubTask = (taskId, subTaskId, newText) => setTasks( Immer.produce(
	tasks => tasks[taskId][subTaskId].text = newText
))
```

# Drawbacks

- Any change to the syntax of a language initially fragments the community to a certain degree. Early adopters might use the new syntax and initially not be understood by more traditional developers. 
-  Currently "JSX is an XML-like syntax extension to ECMAScript". It is used to template HTML in JS. While JSX is associated with React, it is independent to React and is used elsewhere. The current suggestions is beyond the scope of JSX, and will require distinct file extension. 
- Is it a breaking change? Not necessarily. Typescript added new syntactic features that do not exist in Javascript. So did JSX. However, JS code is syntactically valid in JSX and Typescript. The challenge is to add syntactic features, yet for current JSX syntax to be valid in the new syntax. 
- A new syntax won't be supported initially by external tools, and type systems. 

# Alternatives
- Suggested declarations use a different pattern than JS (`M:` compared to `const`, `let`, and `function`). The different pattern indicates that the suggested declarations are not exactly JS. An alternative would be to use declarations which are more aligned with JS declarations, such as `memo` and `state`. 

## Babel Macros
solution might not come from new syntax, but from existing syntax - from JS to JS transformations. Specifically, developer experience could be improved by leveraging Babel macros. An overall approach can yield most of the benefits presented in this RFC, with a minimal cost. 
1. One memoizing macro - It will simplify memoization in React, and will appropriately transform into React.memo, useMemo or useCallback, and pass dependencies. (named `$` in the example below)
2. A macro which simplifies state management. (named `useStateM` in the example below)

```jsx
import {useStateM, $} from "react/macros"

const CompleteAndShare = ({contactsCompletedStats, contacts}) => {
	const [tasks, setTasks] = useStateM(initialTasks)

	const toggleCompleted = $( (id) => setTasks(tasks[id].completed = !tasks[id].completed ) )
	
	const editSubTask = $( (taskId, subTaskId, newText) => setTasks(tasks[taskId][subTaskId].text = newText ))

	const completedTasks = $( tasks.reduce(
		(accum, task) => accum+task.completed,
		0
	))

	const shareCompletedTasks = $( (contactId) => API.shareCompleted(contactId, completedTasks) )

	return (
		<>
			<h1>Complete and share</h1>
			<ControlPanel shareCompletedTasks={shareCompletedTasks} contacts={contacts} />

			<ContactStats contactsCompletedTasks={contactsCompletedTasks} />
			<SelfStats completedTasks={completedTasks} />
			
			<TaskList tasks={tasks} toggleCompleted={toggleCompleted} editSubTask={editSubTask} />
		</>
	)
})

export default $(CompleteAndShare)
```

# Adoption strategy

More discussion, and a better vision of the API are required before considering adoption strategy. 

# How we teach this

This RFC will actually allow an easier and smoother entry to React. A new developer will no longer need to master 
- how to correctly update state in an immutable way. 
- various memoization tools. 

Documentation could put less emphasis on the above, and instead: 
- Present the new syntactic tools, and how to use them. 
- Explain when it makes sense to memoize.

# Unresolved questions
- How to pass state values and state setters to child components, and to functions. 
