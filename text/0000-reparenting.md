- Start Date: 2018-03-11
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

When writing a component that contains a set of large subtrees that stay relatively the same, but are simply moved around such that React's virtual DOM diffing can't detect the movement, React will end up recreating huge trees it should simply be moving.

The goal of this RFC is to introduce a "Reparent" API that allows portions of the React tree to be marked in a way that allows React to know when to move them from one part of the tree to another instead of deleting and recreating the DOM and component instances.

# Basic example

## Layout
Using reparents to render a page layout that can change structure between desktop and mobile without causing the page contents and sidebar to be recreated from scratch.
```js
class Layout extends PureComponent {
    header = React.createReparent(this);
    content = React.createReparent(this);
    sidebar = React.createReparent(this);

    render() {
        const {isMobile, children} = this.props;

        const header = this.header(
            <div className='header'>
                <Logo />
                <NavigationBar />
            </div>
        );

        const sidebar = this.sidebar(
            <div className='sidebar'>
                <SidebarContent />
            </div>
        );

        const content = this.content(
            <Fragment>
                children
            </Fragment>
        );

        if ( isMobile ){
            return (
                <div>
                    {header}
                    {content}
                    {sidebar}
                </div>
            );
        } else {
            return (
                <div>
                    {header}
                    <div>
                        {content}
                        {sidebar}
                    </div>
                </div>
            );
        }
    }
}
```

## Dynamic template
Using reparents in a dynamic template/widget structure to avoid recreating widgets from scratch when they are move from one section of the template to another.

```js
const TemplateWidgetReparentContext = React.createContext({});

class ReparentableTemplateWidget extends PureComponent {
    static getDerivedStateFromProps(nextProps, prevState) {
        if ( !prevState.widget || prevState.widgetType !== nextProps.widget ) {
            const WidgetClass = getWidgetClassOfType(nextProps.widget);
            return {
                widgetType: nextProps.widget,
                widget: new WidgetClass(),
            };
        }
    }

    render() {
        const {widget} = this.state;

        return (
            <div>
                {widget.render()}
            </div>
        );
    }
}

/**
 * This wrapper uses reparents from Template and the widget's own id
 * to ensure that when a widget is moved from one section to another
 * it is moved by react instead of recreated.
 *
 * This could be turned into an HOC
 */
class TemplateWidget extends Component {
    render() {
        const {id} = this.props;

        return (
            <TemplateWidgetReparentContext.Consumer>
                {templateWidgetReparents => templateWidgetReparents[id](
                    <ReparentableTemplateWidget {...this.props} />
                )}
            </TemplateWidgetReparentContext.Consumer>
        );
    }
}

class Section extends PureComponent {
    render() {
        const {title, widgets} = this.props;

        return (
            <div>
                <h2>{title}</h2>
                {widgets.map(widget => (
                    <TemplateWidget {...widget} key={widget.id} />
                ))}
            </div>
        );
    }
}

class Template extends PureComponent {
    static getDerivedStateFromProps(nextProps, prevState) {
        if ( nextProps.sections === prevState.sections ) return;

        let templateWidgetReparents = prevState.templateWidgetReparents;

        const widgetIds = new Set();
        nextProps.sections.forEach(section => section.widgets.forEach(widget => widgetIds.add(widget.id)));

        return {
            widgetIds: widgetIds.toArray(),
        };
    }

    templateWidgetReparents = {};

    render() {
        const {sections} = this.props;
        const {widgetIds} = this.state;

        for ( const id of widgetIds ) {
            if ( !(id in templateWidgetReparents) ) {
                this.templateWidgetReparents = Object.assign(this.templateWidgetReparents, {
                    [id]: React.createReparent(this),
                });
            }
        }

        for ( const id in this.templateWidgetReparents ) {
            if ( !widgetIds.has(id) ) {
                this.templateWidgetReparents[id].unmount();
                this.templateWidgetReparents = omit(this.templateWidgetReparents, id);
            }
        }

        return (
            <TemplateWidgetReparentContext.Provider value={this.templateWidgetReparents}>
                {sections.map(section => {
                    <Section {...section} key={section.id} />
                })}
            </TemplateWidgetReparentContext.Provider>
        );
    }
}

let templateData = {
    sections: [
        {
            id: 's-1',
            title: 'Untitled 1',
            widgets: [
                {
                    id: 'w-1',
                    widget: 'paragraph',
                    props: {
                        text: 'Lorem ipsum...'
                    }
                }
            ]
        },
        {
            id: 's-2',
            title: 'Lorem ipsum',
            widgets: []
        }
    ]
};

ReactDOM.render(<Template {...templateData} />, app);

// Modify the template to move widget:w-1 to section s-2, it should not be necessary to re-create widget w-1 from scratch
templateData = {
    sections: [
        {
            ...templateData.sections[0],
            widgets: []
        },
        {
            ...templateData.sections[1],
            widgets: [
                templateData.sections[0].widgets[0]
            ]
        }
    ]
};
ReactDOM.render(<Template {...templateData} />, app);
```

# Motivation

Up till now the only way to handle reparenting has been using `ReactDOM.unstable_renderIntoContainer` or Portals to separate a dom node from react rendering and allow that node to be moved around. However this is a complex hack that only works with ReactDOM, it does not work with isomorphic React web apps or other environments like React Native.

This reparenting issue have been left alone for awhile. We have an RFC process now. An RFC for a new context API has made an unstable feature of React stable. And accepted RFCs like createRef have given suggestions on what APIs fitting of React might look like. I think now is a good time to try tackling reparenting.

More motivation can be found in past discussions on reparenting:

- [Support for reparenting (facebook/react#3965)](https://github.com/facebook/react/issues/3965)
- [A gist that has hosted a lot of reparenting discussion](https://gist.github.com/chenglou/34b155691a6f58091953)

# Glossary

- **state tree**: The state tree refers to both the tree of DOM nodes or other tree of native elements React is rendering to (for React Native and 3rd party environments) and the tree of React Components that allows React to reuse an instance of a Component on a future render.
- **Reparent owner**: A reparent owner is the component passed to the `React.createReparent(component)` function. The owner does not have to be the one rendering the Reparent, you can pass the reparent down for a descendant to render, but it cannot be passed upwards to be rendered by a parent or cousin of the Reparent owner (notwithstanding use of Portals).

# Detailed design

This proposal introduces a `React.createReparent(component)` function. It accepts a component instance (typically `this`) which is the reparent owner and returns a *Reparent*. The "Reparent" naming is open to better name proposals.

A Reparent is tied to its owner, when its owner is unmounted the reparent is also unmounted and the detached state tree is discarded. This allows reparents to hold detached state trees without doing so in a permanent way that would leak memory.

A Reparent has an additional `.unmount()` method, this can be used to force the reparent to unmount and discard its state tree like it would when it's owner unmounts, even when the owner is not unmounting. This allows reparents to be hoisted upwards by registering the Reparent in a parent Component and then passing the Reparent down to children through props or context for a descendant to render. And allows them to be used for dynamic content and explicitly unmounted when necessary (the descendant no longer needs the Reparent but the Reparent owner will not be unmounting). See the "Dynamic template" example.

The Reparent itself is a function that accepts children for the reparent. The return value is a special Fragment used to render the reparent and its contents somewhere in the state tree.

When React intends to remove the dom/native tree rendered from this special fragment because it is not present in a later render, React instead detaches the tree and holds a reference to it in the Reparent instead of purging it. If a fragment returned by the reparent later is rendered by a component the detached dom/native tree is re-used.

In addition to retaining the dom/native tree for future use the rest of the state tree belonging to the Reparent is also retained. Such that React Component instances present in a Reparent are not unmounted until the Reparent is rendered without them (explicitly discarding the component), the Reparent is unmounted via `.unmount()` (explicitly destroying the Reparent and its whole tree), or the Reparent is implicitly unmounted when its owner unmounts (implicitly destroying the Reparent and its whole tree).

Unlike a normal fragment the fragment returned by a reparent may only be used once in the entire state tree. If a fragment returned by a Reparent is used multiple times only the latest one will be rendered (they will not be duplicated as normal in React) and during development React may emit warnings that a reparent is used in multiple spots like the warnings when a duplicate key is found.

When React DOM intends to detach a DOM node (because the Reparent from a previous call to render is no longer returned in the virtual DOM for a later render), it may use `removeChild` to detach the DOM node, but must retain a reference to it so it may be re-inserted later if the Reparent is rendered in a future render. However when React DOM intends to move a DOM node (because the Reparent from a previous calls to render has a different parent in the virtual DOM on a later render) React DOM should avoid using `removeChild` and instead just call `appendChild`/`insertChild` without first calling `removeChild` (unless it detects some sort of situation like a reference cycle that would make this impossible). This is an attempt to avoid some of the side effects of Reparenting, in particular it is to avoid IE11's behaviour of pausing a `<video>` when you `removeChild` then `appendChild` instead of just using `appendChild`.

The fragment returned has an implicit key which is unique to the Reparent. A Reparent itself is like a key that is unique beyond just a single element's children so there is no need for the user to specify an additional key to use its contents in an array.

Naively, createReparent without the reparenting and unmount behaviour behaves similar to the following implementation:
```js
React.createReparent = function() {
    const key = generateUniqueKey();

    const Reparent = function(children) {
        return React.createElement(Fragment, {key}, children);
    };
    Reparent.unmount = () => {
        throw new Error('Not implemented');
    };

    return Reparent;
};
```

And conforms to the following types/interface:
```js
type ReparentFunction = (ReactNodeList) => ReactFragment;
type ReparentObject = {
    unmount(): void;
};
type Reparent = ReparentFunction & ReparentObject;

interface React {
    createReparent(Component): Reparent;
}
```

# Limitations

Some DOM nodes have quirks they exhibit when they are moved from one parent DOM node to another parent DOM node. These quirks are unavoidable and solving them is not part of the reparenting RFC. The benefits of having the option to reparent nodes if desired is still significant even given the limitations. Some of these quirks may be fixed by either making the state controlled (see [registered prop as ref](https://github.com/reactjs/rfcs/pull/28) for a possible solution for this) or by storing the state before the reparent and restoring it afterwards (see [getSnapshotBeforeUpdate](https://github.com/reactjs/rfcs/pull/33) for a possible solution).

- `<video>`/`<audio>` elements may pause (Chrome exhibits this behaviour; IE11 only exhibits this behaviour if you `removeChild` before you `appendChild`/`insertChild`; Firefox and Safari do not exhibit this quirk)
- `<iframe>` elements will refresh, completely losing their state. This is unavoidable. However, iframes have very limited use cases so those making use of iframes should be aware that they cannot move their iframes around unless they accept that it will refresh when they do.
- `.swf` Flash players will reload. (*unconfirmed*)
- The currently focused element may lose its focus if it is being moved. (*unconfirmed*)
- Scrollable containers will lose their scroll position and reset to the initial scroll position when they are moved.

# Drawbacks

The `.unmount()` method is not necessarily perfect in identifying when a tree is detached but kept or discarded and to be unmounted. If someone chooses to use createReparent dynamically and forgets to call `.unmount()` they will leak memory.

# Alternatives

## createKey

@gaearon proposed [a createKey API](https://gist.github.com/chenglou/34b155691a6f58091953#gistcomment-1460942)

```js
class MyComponent extends Component {
    contentKey = React.createKey(this);

    render() {
        const {isB, children} = this.props;

        const content = <Fragment key={this.contentKey}>{children}</Fragment>;

        return (
            <div>
                {isB && content}
                <div>{!isB && content}</div>
            </div>
        );
    }
}
```

The createKey API could work, however it has some limitations that createReparent does not:

- While createKey and createReparent shares the same advantage that unmounting of its host results in unmounting of the reparentable root. contentKey does not have a secondary method of unmounting/discarding the tree. React cannot differentiate between a detached tree and a discarded/unmountable tree. As a result it cannot be used for varying numbers of reparentable roots as trees for discarded keys remain in memory as leaks.
- createKey adds alternative behaviour using just the `key`, createReparent gives React internals more control in how it they decide to handle the link between the tree and the reparent.

## DetatchedTree

If `.unmount()` proves to be too complex it may be possible to make detached trees work with createReparent, createKey, or other global key methods by using a Fragment-like `<DetatchedTree />` component which holds a reference to the detached tree in the React tree but omits it from the DOM. Then React knows it may unmount components and trees if it is not used as part of the live tree or in a DetatchedTree.

```js
class MyComponent extends Component {
    contentKey = React.createKey();

    render() {
        const {show, children} = this.props;

        const content = <Fragment key={this.contentKey}>{children}</Fragment>;

        return (
            <div>
                <DetatchedTree>
                    {!show && content}
                </DetatchedTree>
                {show && content}
            </div>
        );
    }
}
```

# Adoption strategy

Reparents can be released in a feature release of React, there are no breaking changes.

If people are still using some experimental 3rd party libraries like [react-teleporter](https://github.com/jaredly/react-teleporter) that use `ReactDOM.unstable_renderIntoContainer` or Portals to hack reparenting into React DOM, we may wish to provide migration guides to the Reparent API or update these libraries to make use of the Reparent API.

# How we teach this

"Reparents" may not be the best terminology, some terminology including "root" in the name to refer to the segment of the state tree created by a reparent that can be moved.

We may need a new documentation page to explain reparenting as one was added for portals. Like portals this is an advanced feature that does not need to be part of the main tutorials.

# Unresolved questions

- How do reparent instances interact with hydration?
  - Perhaps when hydrating the dom tree, portions of the dom tree that match up with portions of the virtual dom belonging to a reparent will be given to the Reparent to hydrate as its dom tree.
- Should the Reparent function just accept a single children argument, or should it accept ...children rest and pass it on to createElement to behave similar to createElement.
- If a Reparent is detached and it's DOM tree is detached from the document, should it render the children it is passed in the Reparent function; or should it save the most recent children value passed to it and wait till it is re-mounted before actually rendering those children?
