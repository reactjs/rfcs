- Start Date: 2021-09-27
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

An Object Oriented handler hook. As alternative (not to replace) to useReducer hook. 

Maintain a unique instance of the handler class instance on memory. Heavy functions are not instantiated in every render. Bring back object oriented advantages, leaving behind some React overhead like useCallback, useReducer and custom hooks.

# Basic example

Real life like example from a certificate issuing module.

ModalCert.tsx:
```jsx
import { CertHandler } from './CertHandler';
import { useStateHandler } from 'use-state-handler';

const ModalCert: FunctionComponent<ModalProps> = ({ cert_name }) => {

  const [ cert, handler ] = useStateHandler( CertHandler );

  useEffect( () => { handler.load( cert_name ) }, [] )

  return (
    <div className="box">

      <input
        type='text'
        placeholder="ej: Jhon"
        value={cert.name}
        name="name"
        onChange={ handler.setInput }
      />
      
      <input
        type='text'
        placeholder="ej: Jhon@mail.com"
        value={cert.mail}
        name="mail"
        onChange={ handler.setInput }
      />

      <select
        name="type"
        onChange={ handler.setType }
      >
        <option value="A"></option>
        <option value="B"></option>
      </select>

      <span className="subtitle">Output Signature:</span>
      <div className="signature"> cert.description </div>

      <MyControlComponent 
        certHandler={handler}
      />

      <button onClick={handler.save}>
        Save
      </button>

    </div> 
  );

}

```

CertHandler.ts
```jsx
import { StateHandler } from 'use-state-handler';

class CertHandler extends StateHandler<ICert> {

  private descA = " Description A ...";
  private descB = " Description B ...";

  static initState = { name : "", mail : "" }

  instanceCreated = ( ) => 
    this.load( "" )


  public load = ( name : string ) =>
    fetch( "url", name )
      .then( data => setState( data ) )
  

  public setValue = ( name: string, val : any ) => {
    this.setState (  {...this.state, ...{ [name] : val } }  )
  }

  public loadControlOptions = ( id : string ) => 
    fetch( "url_details" )
      .then( data => setState( {...this.state, ...{ details : data } } )

  public setInput = ( e : ChangeEvent<HTMLInputElement> ) => {   
    this.setValue ( e.target.name, e.target.value )
  } 

  public setType = ( e : ChangeEvent<HTMLSelectElement> ) => {   
    let type = e.target.value;
    let desc : string
    
    if(type === "A")
      desc = state.name + this.descA;
    else
      desc = this.descB;

    setState({...this.state, ...{ type : type, description : desc } });
  } 


  save = () =>
    post( "save_url", this.state )

}

```
NewModalCert.tsx:
```jsx
import { CertHandler } from './CertHandler';
import { useStateHandler } from 'use-state-handler';

const NewModalCert: FunctionComponent<ModalProps> = ({ cert_name }) => {

  const [ cert, handler ] = useStateHandler( CertHandler, CertHandler.initState );


  return (
    <div className="box">

      <input
        type='text'
        placeholder="ej: Jhon"
        value={cert.name}
        name="name"
        onChange={ handler.setInput }
      />
      
      <input
        type='text'
        placeholder="ej: Jhon@mail.com"
        value={cert.mail}
        name="mail"
        onChange={ handler.setInput }
      />

      <select
        name="type"
        onChange={ handler.setType }
      >
        <option value="A"></option>
        <option value="B"></option>
      </select>

      <span className="subtitle">Output Signature:</span>
      <div className="signature"> cert.description </div>

      <MyControlComponent 
        certHandler={handler}
      />

      <button onClick={handler.save}>
        Save
      </button>

    </div> 
  );

}

```

# Basic example 2

Shows handlers inter-operation. Successfully trigger render from another state handler.

App.tsx:
```jsx
function App() {

  const [ books, booksHandler ]           = useStateHandler( BooksHandler );
  const [ form, formHandler]              = useStateHandler( FormHandler )

  useEffect(() => {
    booksHandler.load();
    formHandler.init();
    booksHandler.resetForm = formHandler.init;
  }, [ /*prestamosHandler, formHandler*/ ]);

  return (
    <>
      <Menu />
      <div className="section">
        <div className="container page">

          <Item type='add' book={form} setValues={formHandler.setValue} save={booksHandler.saveItem}>
            Add
          </Item>

          {books?.map( p => <Item key={p.id} save={booksHandler.saveItem} book={p}>Edit</Item> )}
       
        </div>
      </div>
    </>
  );
}

export default App;
```

# Motivation


This hook is suited for certain uses:

* When state logic needs multiple methods that otherwise will need a complex useReducer or complex custom hooks. This alternative maintain code simple, clean and encapsulated.  
* When you want avoid the use of useCallback, useRef and other react-overhead-like solutions.
* Actually implemented and on production environments, this hook has simplified coding on the team.

Note that useState, useReducer and custom hooks are wide used alongside this hook. Is a complement to existing basis.

**Why to integrate to react core and RFC process?**

* I don't know about react source code but I konw this can be improved in many ways; from definition to implementation.
* Eliminate undesired warnings about handlers in dependency array list.


# Detailed design

The first version was very simple:

```jsx
function useStateHandler<H extends StateHandler<T>, T>( handlerClass : new ( s : React.Dispatch<React.SetStateAction<T>>) => H, initial_value: T | (() => T)) : [T, H]  {
  const [st, setSt]                 = React.useState<T>( initial_value );  
  const [handler, setHandler ]      = React.useState<H>( () => new handlerClass( setSt ) );
  
  handler.state = st;

  return [ st, handler ];
}

abstract class StateHandler<T> {
  
  state?            : T;
  setState          : React.Dispatch<React.SetStateAction<T>>; 

  constructor( sts : React.Dispatch<React.SetStateAction<T>> ) { this.setState = sts }

}

```


But it has taken some minor changes.

Actual source code can be found here:
https://github.com/ksoze84/useStateHandler/blob/main/src/main.ts


# Drawbacks

As an optional feature doesn't have any major drawbacks except for adding things to learning for beginners.

# Alternatives

* useReducer
* useCallback
* other libraries like redux.

# Adoption strategy

Please review, change if necesary. For example, I'm not not convinced on the actual definitions for initial state in the hook.

# How we teach this

What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing React patterns?

* Class or Object handler hook. 
* StateHandler superclass
* Takes back the this.state and this.setState method. 

Would the acceptance of this proposal mean the React documentation must be
re-organized or altered? Does it change how React is taught to new developers
at any level?

* Yes, documentation should be augmented. 
* I think, if this hook is used, that can simplify some functional components concepts 

How should this feature be taught to existing React developers?

* Like a new feature to simplify certain problems asociated with complex state management in functional components

# Unresolved questions

Optional, but suggested for first drafts. What parts of the design are still
TBD?

* Initial state definition. Now can be defined -perhaps- in too many ways. This may be confusing.
