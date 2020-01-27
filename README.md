# react-vrpc
React wrapper around the [vrpc](https://github.com/bheisen/vrpc) library

## Step 1 - Create VrpcProvider

In the `index.js` of your react app add something like:

```javascript
import { createVrpcProvider } from 'react-vrpc'

const VrpcProvider = createVrpcProvider({
  domain: '<domain>',
  broker: 'wss://vrpc.io/mqtt',
  backends: {
    myBackend: {
      agent: '<agentName>',
      className: '<className>',
      instance: '<instanceName>',
      args: ['<constructorArg1>', '<constructorArg2>', ...]
    }
  },
})
```

You can use any number of backends with VRPC by adding several objects under
the `backends` property. Refer to them later simply by their name
(here: `myBackend`).

Think of `myBackend` as if it was a remotely available
instance of the class you chose in the `className` property.

Depending on your backend architecture *react-vrpc* allows you to:

1.  Create an (anonymous) instance of a class

    > define parameters: `agent`, `className`, `args`

2.  Create (if not exists) and use a named instance of class

    > define parameters: `agent`, `className`, `instance`, `args`

3.  Use (never create) an existing named instance of a class

    > define parameters: `agent`, `className`, `instance`

4.  Use all named instances of a class

    > define parameters: `agent`, `className`

    In this case your backend object reflects an array of proxy instances, and
    you can access their id using `<proxy>._id` member variable.

    You may also combine instances of different classes by using
    a regular expression in the `className` property. The class name of each
    instances is accessible via the `<proxy>._className` member variable.

> **TIP**
>
> If your backend instance is capable to emit events you can automatically
> subscribe to those by configuring an additional `events` property:
>
> ```javascript
> const VrpcProvider = createVrpcProvider({
>   backends: {
>     myInstances: {
>       agent: '<agentName>',
>       className: '<className>',
>       events: ['<eventName1>', '<eventName2>', ...]
>     }
>   },
> })
> ```
> Corresponding notifications will result in a component update (see below) and
> the event payload is accessible via the `<proxy>.<eventName>` property.
>

## Step 2 - Use the VrpcProvider

At a top-level position of your component hierarchy add the `<VrpcProvider>`

```javascript
ReactDOM.render(
  <VrpcProvider username='test' password='secret'>
    <Router>
      <Switch>
        <Route exact path='/myComponent' component={MyComponent} />
      </Switch>
    </Router>
  </VrpcProvider>
  document.getElementById('root')
```

> **NOTE**
>
> If working with `https://vrpc.io` as broker solution you may also
> use `token` instead of `username` and `password`.

## Step 3 - Give a component access to backend functions

```javascript
import React from 'react'
import { withVrpc } from 'react-vrpc'

class MyComponent extends React.Component {

  async componentDidMount () {
    const { myBackend, vrpc } = this.props
    const ret = await myBackend.aBackendFunction('test')
  }
}

export default withVrpc(MyComponent)
```

> **NOTE 1**
>
> Optionally, you can use the `vrpc` object which reflects the remote proxy
> instance.

> **NOTE 2**
>
> Always use `await` as all VRPC calls need to travel the network and are
> asynchronous by default. If the backend function you are calling is
> `async` itself, simply add a second `await`, like so:
>  ```javascript
> const ret = await await myBackend.anAsyncBackendFunction('test')
> ```
> You can use simple `try/catch` statements, VRPC forwards potential exceptions
> on the backend for you.

> **NOTE 3**
>
> You may select only one or a subset of your backends for a specific component.
> Simply provide a string or an array of strings specifying the
> corresponding backends as first argument of the `withVrpc` function.
> ```javascript
> export default withVrpc('backend1', MyComponent)
> ```
> This will limit the re-rendering of `MyComponent` solely to changes observed
> in `backend1` drastically improving performance when using several backends.

For all details, please visit: https://vrpc.io, or
https://github.com/bheisen/vrpc


## Step 4 - Update component upon backend changes

This step is optional and typically only needed if you configured your
backend to reflect an array of instances (see point 4 of potential usages).

There are two reasons why such an update may be triggered:

1.  The number of instances changed (reduced or increased)
2.  An event was emitted by one of the instances and the corresponding
    value of the `<proxy>.<eventName>` member changed.

For both cases you can use the `componentDidUpdate(prevProps)` life-cycle
method of react:

```javascript
async componentDidUpdate(prevProps) {
  // myInstances reflects a backend with multiple instances
  const { myInstances } = this.props
  const prevInstances = prevProps.myInstances
  if (prevInstances.length !== myInstances.length) {
    // update needed, i.e. call setState in this component
  } else {
    // let 'state' be a subscribed event
    const changed = prevInstances.filter(({ state }, index) => (
      state !== myInstances[index].state
    ))
    if (changed.length > 0) {
      // updated needed, at least on state change detected
    }
  }
}
```
