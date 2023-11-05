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
    header = React.createReparent();
    content = React.createReparent();
    sidebar = React.createReparent();

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
                {children}
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

## Detach trees of dom nodes / components

```js
class Foo extends Component {
    componentDidMount() {
        console.log('Mounted');
    }
    componmentWillUnmount() {
        console.log('Unmounted');
    }
    render() {
        return null;
    }
}

class DetachableTree extends Component {
    reparent = React.createReparent();

    render() {
        const content = this.reparent(this.props.children);
        return this.props.show && content;
    }
}

ReactDOM.render(
    <DetachableTree show={true}>
        <Foo key={1} />
    </DetachableTree>,
    container);
// log: Mounted

// Children can be detached from the dom tree with `show = false`
ReactDOM.render(
    <DetachableTree show={false}>
        <Foo key={1} />
    </DetachableTree>,
    container);
// But they will not actually be unmounted

// If you set `show = true` later, the previously rendered state tree will be re-inserted
ReactDOM.render(
    <DetachableTree show={true}>
        <Foo key={1} />
    </DetachableTree>,
    container);

// The state tree of the reparent is what is retained, not the contents.
// So if you change the contents of the DetachableTree, then components unmount as normal
ReactDOM.render(
    <DetachableTree show={true}>
        <Foo key={2} />
    </DetachableTree>,
    container);
// log: Unmounted
// log: Mounted

// But if you unmount the DetachableTree the reparent will unmount
// and all of the normal react elements inside will also unmount.
ReactDOM.render(
    <div />,
    container);
// log: Unmounted
```

## Table with movable cells

Using dynamic reparents within a table to ensure that cells are not recreated from scratch when moved from one row to another.

```js
class TableWidget extends PureComponent {
    state = {
        table: {
            cols: 4,
            rows: 0,
            cells: [],
        },
    };

    addRow() {
        this.setState(state => ({
            table: update(state.table, {
                rows: state.table.rows + 1,
                cells: {
                    [state.table.rows + 1]: {
                        $set: new Array(state.table.cols).fill(null),
                    },
                },
            })
        }));
    }

    addData(row, col, data) {
        this.setState(state => ({
            table: update(state.table, {
                cells: {
                    [row]: {
                        [col]: {
                            $set: {
                                id: uuid(),
                                data,
                            },
                        },
                    },
                },
            })
        }));
    }

    moveData(from, to) {
        this.setState(state => {
            let {table} = state;
            // Set the to cell to the from cell's data
            table = update(table, {
                cells: {
                    [to.row]: {
                        [to.col]: {
                            $set: table.cells[from.row][from.col],
                        },
                    },
                },
            });
            // Set the from cell to null
            table = update(table, {
                cells: {
                    [from.row]: {
                        [from.col]: {
                            $set: null,
                        },
                    },
                },
            });
            return {table};
        });
    }

    removeData(row, col) {
        this.setState(state => {
            // Set the cell to null
            let {table} = state;
            table = update(table, {
                cells: {
                    [row]: {
                        [col]: {
                            $set: null,
                        },
                    },
                },
            });
            return {table};
        });
    }

    // Dynamic store for reparents, the reparents are created as-needed for cells in the table
    cells = Object.create(null);
    getCell = (cell, row, col) => {
        this.cells[cell.id] = this.cells[cell.id] || React.createReparent();
        return this.cells[cell.id](<Cell row={row} col={col} cell={cell} />);
    };

    render() {
        const {table} = this.props

        return (
            <table>
                {table.cells.map((cols, row) => (
                    <tr key={row}>
                        {row.cols.map((cell, column) => (
                            <td key={column}>
                                {this.getCell(cell, row, column)}
                            </td>
                        ))}
                    </tr>
                ))}
            </table>
        );
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

        const widgetIds = new Set();
        nextProps.sections.forEach(section => section.widgets.forEach(widget => widgetIds.add(widget.id)));

        const templateWidgetReparents = {};

        for ( const id of widgetIds ) {
            if ( prevState.templateWidgetReparents && prevState.templateWidgetReparents[id] ) {
                templateWidgetReparents[id] = prevState.templateWidgetReparents[id];
            } else {
                templateWidgetReparents[id] = React.createReparent();
            }
        }

        return {
            sections: nextProps.sections,
            templateWidgetReparents,
        };
    }

    render() {
        const {sections, templateWidgetReparents} = this.state;

        // Make sure React detaches trees for reparents we are still using instead of unmounting them
        for ( const reparent of Object.values(templateWidgetReparents) ) {
            reparent.keep();
        }

        return (
            <TemplateWidgetReparentContext.Provider value={templateWidgetReparents}>
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

// Modify the template to move widget:w-1 to section s-2,
// it should not be necessary to re-create widget w-1 from scratch
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

## Goals of reparenting

1. Ensure that react instance state (`state` and `this.*`) is not destroyed when reparenting.
2. Avoid the high cost of re-render when React could just move a component from one parent to another.
3. Where possible, retain the state of the underlying element external to React: like video buffers, state contained in custom-element instances, 3rd party library annotations (e.g. jQuery's `data` storage), and other native state.

1 and 2 have a high enough value that it's ok for 3 to not function completely, where incomplete means:

- The platform is non-dom and may not actually have a dom-like concept of moving a view from one parent to another.
- Some of the state in the native element is lost when we use appendChild like focus, scroll, playing state (see [Limitations](#limitations)).

More motivation can be found in past discussions on reparenting:

- [Support for reparenting (facebook/react#3965)](https://github.com/facebook/react/issues/3965)
- [A gist that has hosted a lot of reparenting discussion](https://gist.github.com/chenglou/34b155691a6f58091953)

# Requirements

There are a variety of use cases for reparenting. Some of them have certain requirements of the API that not all suggestions cover.

1. It must not rely on user-provided strings.

   We already use string keys for local key comparisons. This works fine for local keys, but guaranteeing the application wide uniqueness of a string should not be a requirement of using a reparent. No other React API works this way and it adds difficulty to usage of reparents within libraries.

2. It must be possible to detach a tree and reuse it later.

   Sometimes you want to detach a tree and hold on to it instead of just moving it around. For example, it may be beneficial to remove a modal's contents when not in use. Or the detached nature may simply be a side effect of complex application logic, perhaps involving async and nested components, which results in it being difficult to guarantee that a reparent is present somewhere in the react state tree. Thus any implementation must provide enough information for React to differentiate between when a reparent is not present in a tree but has just been detached; and when a reparent is no longer in use by its owner and should be unmounted.

3. It must be possible to create reparents dynamically.

   Sometimes instead of a reparent being used for a known tree that you just want to move, it is instead used for dynamic content that comes from some store/database. You need to be able to move the content generated from this from one part of the state tree to another. But any number of these may exist at once.

   Thus it must not be a requirement that reparents be known ahead of time. Additionally, because reparents for dynamic content can be removed dynamically it must be possible to unmount any reparent no longer in use, without the owner of the reparent itself having to unmount.

4. It must be possible to use a reparent in the render method of a component that is a descendent of the actual owner.

   Special use cases the dynamic template example and common use cases like Drag & Drop often may involve moving a reparent from one React component to another React component. Thus it must be possible for the reparent's owner to be a React component higher up in the state tree that is able to pass its reparents down to children (through props or context) and have those children render the reparent.

5. Requirements 2, 3, and 4 interact specially and a proper reparent implementation must satisfy all of them at the same time.

   - Unmounting in React involves the calling of lifecycle methods that may involve cleanup of possible global subscriptions (event listeners, data observers, etc) which may leak if not done. As a result we cannot simply store a reference to the state tree in a reparent and let the state tree be GCd when the reparent is no longer being referenced by the code that created it. React must hold a reference to the state tree and explicitly unmount it when not in use.
   - Reparents may be created/destroyed dynamically and be owned by a component that may never unmount. So we need to know when we can unmount a reparent, separate from just when its owner unmounts.
   - We need detached trees, so we cannot simply unmount any reparent that is no longer present in the state tree.
   - A reparent may be rendered in a component other than its owner. Thus we cannot simply unmount a reparent when it has not been used in the render method of its owner.

   As a result, in addition to implementing the global-ishly unique key behaviour needed to move reparents any implementation also needs to provide an explicit declaration that can be used to differentiate detached from unused reparents.

6. It cannot require React DOM's Portals.

   Portals only work in the client side DOM code. They don't work on the server, if you render something isomorphic and then put it into a portal on the client it will be recreated from scratch. And Portals are not available in non-dom environments like React Native. While native doesn't necessarily need reparenting for native views, reparenting itself is still useful as a way of keeping the state of React components without resorting to moving absolutely everything into Flux/Redux.

# Glossary

- **state tree**: The state tree refers to both the tree of DOM nodes or other tree of native elements React is rendering to (for React Native and 3rd party environments) and the tree of React Components that allows React to reuse an instance of a Component on a future render.
- **global-ish key**: The global-ish key nature of reparents refers to how normally in react the string `key="..."` is only locally unique within its parent, but a reparent must be unique within any parent. Generally this would be though of as "globally unique", however because reparents have an owner they only actually need to be unique within the portion of the state tree under their owner.

# Detailed design

This proposal introduces a `React.createReparent()` function which returns a new *Reparent*. The "Reparent" naming is open to better name proposals.

The Reparent itself is a function that accepts children for the reparent. The return value is a special Fragment used to render the reparent and its contents somewhere in the state tree.

A Reparent has an additional `.keep()` method that acts like calling the Reparent function but without modifying children or returning the fragment. This function is used to allow components to tell reparents to hold references to state trees without actually rendering them and without doing so in a permanent way that would leak memory. This allows reparents to be hoisted upwards by registering the Reparent in a parent Component and then passing the Reparent down to children through props or context for a descendant to render. And allows them to be used for dynamic and async content where the reparent may not be rendered for a moment. See the "Dynamic template" example.

When the Reparent function or its `.keep()` is called in a `render()` React notes that the component the `render()` belongs to "has a reference" to the reparent.

At the end of a render if no `render()` "has a reference" to the reparent then React will discard the state tree and unmount any components inside the reparent's children as normal.

At the end of a render if at least one `render()` "has a reference" to the reparent but a fragment created by the reparent function is not present in the the result of any `render()` instead of discarding the state tree React will *detach* the state tree and keep a reference to it. React components in the children will not be unmounted. If the native tree can reasonably detached the renderer should make an effort to detach the tree. If the native tree does not have a concept of being detached and re-attached it is acceptable for the native tree to be removed and recreated later.

For react-dom this means using `removeChild` to detach the dom tree and keeping a reference to it in memory to re-insert into the dom later with `appendChild`/`insertChild`. When React DOM intends to move a DOM node (because the Reparent from a previous calls to render has a different parent in the virtual DOM on a later render) React DOM should avoid using `removeChild` and instead just call `appendChild`/`insertChild` without first calling `removeChild` (unless it detects some sort of situation like a reference cycle that would make this impossible). This is an attempt to avoid some of the side effects of Reparenting, in particular it is to avoid IE11's behaviour of pausing a `<video>` when you `removeChild` then `appendChild` instead of just using `appendChild`.

When a reparent is being moved or being detached and later reattached `getSnapshotBeforeUpdate` should be called before the component is moved or detached and `componentDidUpdate` should be called with the snapshot after the component is moved or re-attached. This will allow users to save and restore state that may be discarded by the native implementation when native trees are moved. See [Limitations](#limitations).

When a reparent's children are changed React removes and unmounts unused components as normal. References to children no longer used by the reparent are not retained unless the children are reparents that are themself still referenced by a `render()`.

Unlike a normal fragment the fragment returned by a reparent may only be used once in the entire state tree. If a fragment returned by a Reparent is used multiple times only the latest one will be rendered (they will not be duplicated as normal in React) and during development React may emit warnings that a reparent is used in multiple spots like the warnings when a duplicate key is found.

The fragment returned has an implicit key which is unique to the Reparent. A Reparent itself is like a key that is unique beyond just a single element's children so there is no need for the user to specify an additional key to use its contents in an array.

Naively, createReparent without the reparenting and unmount behaviour behaves similar to the following implementation:

```js
React.createReparent = function() {
    const key = generateUniqueKey();

    const Reparent = function(children) {
        this.keep();
        return React.createElement(Fragment, {key}, children);
    };
    Reparent.keep = () => {
        console.warning('keep: Not implemented');
    };

    return Reparent;
};
```

And conforms to the following types/interface:

```js
type ReparentFunction = (ReactNodeList) => ReactFragment;
type ReparentObject = {
    keep(): void;
};
type Reparent = ReparentFunction & ReparentObject;

interface React {
    createReparent(): Reparent;
}
```

# Limitations

Some DOM nodes have quirks they exhibit when they are moved from one parent DOM node to another parent DOM node. These quirks are unavoidable and solving them is not part of the reparenting RFC. The benefits of having the option to reparent nodes if desired is still significant even given the limitations. Some of these quirks may be fixed by either making the state controlled (see [registered prop as ref](https://github.com/reactjs/rfcs/pull/28) for a possible solution for this) or by storing the state before the reparent and restoring it afterwards using [getSnapshotBeforeUpdate](https://reactjs.org/docs/react-component.html#getsnapshotbeforeupdate) and `componentDidUpdate`'s `snapshot` argument.

- `<video>`/`<audio>` elements may pause (Chrome exhibits this behaviour; IE11 only exhibits this behaviour if you `removeChild` before you `appendChild`/`insertChild`; Firefox and Safari do not exhibit this quirk)
- `<iframe>` elements will refresh, completely losing their state. This is unavoidable. However, iframes have very limited use cases so those making use of iframes should be aware that they cannot move their iframes around unless they accept that it will refresh when they do.
- `.swf` Flash players will reload. (*unconfirmed*)
- The currently focused element will lose its focus if it is being moved.
- Scrollable containers will lose their scroll position and reset to the initial scroll position when they are moved.

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

## .unmount()

Originally this RFC proposed an imperative `.unmount()` in place of `.keep()`. `React.createReparent` would be called with `this` to give the reparent an owner. The reparent would unmount when its owner unmounts or could be explicitly unmounted with `.unmount()`.

This proposal was replaced as `.keep()` has the advantage of not holding on to detached trees if the user forgot to unmount a dynamic reparent. And `.keep()` is compatible with more techniques and use cases.

## DetachedTree

If `.unmount()` proves to be too complex it may be possible to make detached trees work with createReparent, createKey, or other global key methods by using a Fragment-like `<DetachedTree />` component which holds a reference to the detached tree in the React tree but omits it from the DOM. Then React knows it may unmount components and trees if it is not used as part of the live tree or in a DetachedTree.

```js
class MyComponent extends Component {
    contentKey = React.createKey();

    render() {
        const {show, children} = this.props;

        const content = <Fragment key={this.contentKey}>{children}</Fragment>;

        return (
            <div>
                <DetachedTree>
                    {!show && content}
                </DetachedTree>
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