- Start Date: 2024-06-10
- RFC PR:
- React Issue:

# Summary

Reflection lets dynamic applications analyze a given component and extract information about it.
It also enables extracting information about a given application.

# Basic example

```
import {
    isArray,
    hasProp,
} from "react-reflection"

const ParentComponent = ({ children, ...rest }) => {

    const isArray = isArray(children)
    let filteredChildren
    if (isArray)
    {
        filteredChildren = children.filter(i => !hasProp(i, 'superAdmin'))
    }
    else
    {
        filteredChildren = hasProp(children, 'superAdmin') ? null : children
    }

    return <>
        <h1>Parent component</h1>
        {
            isArray
            ?
            <ul>
            {
                filteredChildren.map((child, index) => <li key={index}>{child}</li>)
            }
            </ul>
            :
            children
        }
    </>
}
```

# Motivation

Right now developers have no way to extract information about a passed component.
Knowing about a passed component opens the door for more dynamic systems, yet with less verbosity.

# Detailed design

Let's create a dynamic `<Icon>` component. It can accept a wide range of inputs. Let's support a range of possible inputs:

- Inline SVG: `<Icon show={<svg></svg>} />`
- Third-party component:   
```
import SettingsIcon from "@mui/icons-material/Settings"   
<Icon show={SettingsIcon} />
```
- Custom function components:  
```
const CustomIcon = () => <div>I am a custom icon</div>  
<Icon show={CustomIcon} />
```
- A variable:   
```
let icon
switch (entity.type) {
    case "instructor":
        icon = TeacherIcon
    case "student":
        icon = StudentIcon
}
<Icon show={icon} />
```
- A fragment:
```
const fragment = <>
    <div>Some shape here</div>
    <div>Another shape here</div>
</>
<Icon show={fragment} />
```

Now let's build this component:

```
import {
    isFragment,
    isFunctionComponent,
}
const Icon = ({ show }) => {
    if (isFunctionComponent(show)) {
        const Component = show
        return <Component />
    }
    if (isFragment(show))
    {
        return show
    }
}
```

Reflection has been around in many languages and has proven to be a great tool for making things more dynamic.

These are real requirements we face today and we solve them using tricks:

- What actions are marked with a boolean `superAdmin` attribute? We should not show them if the current user is not a super admin.
- Is this `filters` prop passed a list of filters or a single filter?   
```
const filters = <>
    <DataTypeFilter />
    <ScopeFilter />
</>
const alternativeFilters = <DataTypeFilter />
```
- Does this action have a dialog associated with it? Should I wrap it inside a `<DialogContext.Provider>` or not?
```
var actions = entity => <>
    <Action
        url={`https://example.com/?filter=${entity.key}`}
        icon={SearchIcon}
    />
    <Action
        dialog={Selection}    
    />
</>

const Actions = ({ actions }) => {
    // we need to analyze each action and extract data about it
}
```

This is a list of ideas that can be developed inside the `react-reflection`:

- Retrieving information about the app itself => something like getComponents
- Finding a specific component => getComponent("ComponentName")
- Get information about props => getProps(component)
- Get one specific prop => getProp(component, "PropName")
- Check for the presence of a prop => hasProp(component, "PropName")
- Get the list of hooks used in the component => getHooks(component)
- Get the list of states used in the component => getStates(component)
- Get the list of dependencies for a given hook => getDependencies(hook) or getDependencies(component, "HookName")
- Getting the type of an object (not checking if it is something) => getType(object) => These could be Fragment, Array, FunctionComponent, ClassComponent, Hook, State, etc.
- Creating a dynamic instance of a component if possible, without being in DOM => createInstance(component)

# Drawbacks

Since the React team has published the dev tool, the infrastructure for extracting data about a given component is already in the React core.
So, I don't see any drawbacks in this RFC.

# Alternatives

- Becoming less dynamic and more static in terms of meta-component programming
- Using tricks to extract data (diverges from React and slows down the application)
    - Getting the string representation of the component using `.toString()` method and extracting data using regular expressions or string operations
- Creating a custom higher-order component to manage some parts of this. This is real code from our codebase:

```
import React from "react"
import { isSuperAdmin } from "Base"

const Unify = ({
    component,
    ...rest
}) => {

    if (!component) {
        const message = "Component passed to the unify is null or undefined"
        return <span className="hidden">{message}</span>
    }
    if (typeof component === "string") {
        return <>{component}</>
    }
    if (component.props && component.props.superAdmin && !isSuperAdmin()) {
        return <span className="hidden"></span>
    }
    if (component.type) {
        if (typeof component.type === "string") {
            return <>{component}</>
        }
        if (typeof component.type === "function") {
            const Component = component.type
            let props = rest
            if (rest.props) {
                props = rest.props
            }
            return <Component {...component.props} {...props} />
        }
        if (typeof component.type === "string") {
            return <>
                {component.type}
            </>
        }
        if (typeof component.type === "symbol") {
            if (component.type.toString() === "Symbol(react.fragment)") {
                if (component.props && component.props.children) {
                    if (Array.isArray(component.props.children)) {
                        return <>
                            {
                                component.props.children
                                    .filter(i => i !== undefined)
                                    .filter(i => {
                                        if (i.props?.superAdmin === true) {
                                            return isSuperAdmin()
                                        }
                                        else if (
                                            i.type instanceof Function &&
                                            (
                                                i.type.toString().indexOf("superAdmin: true") > 0
                                                ||
                                                i.type.toString().indexOf("superAdmin:!0") > 0
                                            )) {
                                            return isSuperAdmin()
                                        }
                                        else {
                                            return true
                                        }
                                    })
                                    .map((i, index) => {
                                        const { superAdmin, ...rest } = i.props || {}
                                        return <Unify
                                            key={index}
                                            component={
                                                i.type
                                                    ? <i.type {...rest} />
                                                    :
                                                    <Unify
                                                        component={i}
                                                    />
                                            }
                                            {...i.props}
                                            {...rest}
                                        />
                                    })
                            }
                        </>
                    } else {
                        return <Unify
                            component={component.props.children}
                            {...component.props.children}
                            {...rest}
                        />
                    }
                }
            }
        }
    }
    if (typeof component === "function") {
        const Component = component
        return <Component {...rest} {...component.props} />
    }

    return <div>{component}</div>
}

export default Unify
```

# Adoption strategy

There is no change at all. There is no need for codemods. No developer needs to change his knowledge about React if he does not want to use it.

This can even be implemented in another library called `react-reflection`.

# How we teach this

A new section should be added the the React docs. It can go under the `Advanced` menu, or it can be in a top-level `Reflection` menu. It's just added. No other parts of the documentation should be changed.

# Unresolved questions

Optional, but suggested for first drafts. What parts of the design are still
TBD?
