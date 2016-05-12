# Redux Saga Todos Tutorial
In this tutorial we're going to implement [Redux Saga](http://yelouafi.github.io/redux-saga/) in the same [Todos app from the previous document](./react-redux-starter-kit-todos.md). In fact, we're promarily going to be editing [`modules/todos.js`](./react-redux-starter-kit-todos.md#todos-module-srcroutestodosmodulestodosjs) to replace the reducers with sagas.

It's going to be lightly superficial to remove reducers and use sagas instead. The power of sagas doesn't become aparent until you hook it to an API, which we'll be doing in the nest step. This time around we're going to get sagas integrated into our app so that we can focus on the API portion of it later. If you try to use Redux Saga for the first time it can be intimidating because it jumps right into using the API. We're going to work up to it slowly.

If you want to get into Redux Saga, they have a [great beginner's tutorial](http://yelouafi.github.io/redux-saga/docs/introduction/BeginnerTutorial.html). You may also want to read some of their [saga background links](http://yelouafi.github.io/redux-saga/docs/introduction/SagaBackground.html).

We'll cover it more later but Redux Saga makes extensive use of [generator functions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function*). You can read [in depth about generators](http://www.2ality.com/2015/03/es6-generators.html) if you'd like but [the basics](https://davidwalsh.name/es6-generators) become clear quite quickly.


### How do generators work?
Because generators are unfamiliar to most JavaScript developers, much of the documentation above will seem dense and confusing. Don't get discouraged. Generators are really easy.

Test this example out in the online [JavaScript REPL](https://repl.it/CQPk/1)

```js
// A generator function has a star
function * test() {
  // you mark the "steps" with yield
  yield 'hello'
  yield 'goodbye'
  return 'done' // <-- you don't normally return something
}

// calling a generator function returns a generator object
const task = test()

// you "walk" the generator using next
task.next() // --> { value: 'hello', done: false }
task.next() // --> { value: 'goodbye', done: false }
task.next() // --> { value: 'done', done: true }
```

Get it? You "walk" a generator function in "steps". The function "pauses" after each step.

1. Call a generator to get a [Generator object](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Generator).
2. Call [`next()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Generator/next) to step through each [`yield`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/yield)
3. The [`yield`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/yield) keyword means "return and pause", it's why you don't normally see a `return` in generator.
4. You should read about [Iterators and Generators](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Iterators_and_Generators)

### What is redux-saga?
You probably want to read about sagas to get [an authoratative definition](https://msdn.microsoft.com/en-us/library/jj591569.aspx). But in the simplest terms, redux-sagas is a task runner (it literally runs whatever function you give it and goes away). In a similar way to how a reducer is just a *function* that responds to an action, a saga is a *generator* that responds to an action. Although the terminology is incorrect, you can imagine that using redux-saga allows you to *subscribe* to redux to capture asyncronous actions and then *dispatch* synchronous actions later. The classic example is using redux-saga to make a `fetch` request to a server and then dispatch and action to load the data into redux. To do this it utilizes generator functions. If you have been trying to get used to Promises, then you'll get the root concepts very easily. If you've used to callbacks you'll quickly see why this is better.

Another way to think about sagas is as action routers. They're not quite the same as asynchronous reducers because they don't update to the store, they only listen to actions.

Here's some important notes about sagas.

1. Sagas should not update the store directly, that's what a reducer is for.
2. Sagas are for fielding actions *before* they get to reducers, that's why it connects to redux as middleware.
3. Redux-saga uses weird terminology like `put` instead of `dispatch` and `takeEvery` instead of `handle`.
4. Iterating generators is not automatic but redux-saga manages that for you, that's why you normally don't have to call [`next()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Generator/next) on your sagas, redux-sagas does that. Your saga should `yield` a [redux-saga effect](http://yelouafi.github.io/redux-saga/docs/basics/DeclarativeEffects.html).

Here's a generic module that makes use of sagas.

```js
import { combineReducers } from 'redux'
import { createAction, handleActions } from 'redux-actions'
import { takeEvery, delay } from 'redux-saga'
import { put } from 'redux-saga/effects'

// actions and action creators
const MY_ACTION = 'MY_ACTION'
const myAction = createAction(MY_ACTION)

const MY_ASYNC_ACTION = 'MY_ASYNC_ACTION'
const myAsyncAction = createAction(MY_ASYNC_ACTION)

// sagas
function * mySaga ({ payload }) {
  yield delay(1000) // <--- returns an effect that proceeds after a delay
  yield put(myAction(payload)) // <-- returns an effect that calls an action
}

// reducers
const myReducer = handleActions({
  [MY_ACTION]: (store, action) => ({ result: true })
}, {})

// combine sagas
export function * rootSaga () {
  yield [
    // combine all of your module's sagas
    yield * takeEvery(MY_ASYNC_ACTION, mySaga)
    // ^-- calls the myGenerator function on MY_ASYNC_ACTION
  ]
}

// combine reducers
export default combineReducers({
  // combine all of your module's reducers
  myReducer // <-- recieves store.myReducer as store
})
```

## Install Redux Saga
Let's install [redux-saga](https://github.com/yelouafi/redux-saga):

```bash
npm install redux-saga --save
```

## Apply Saga Middleware
Redux Saga is middleware. If you use only Redux Saga for your asynchronous actions you should not need [Redux Thunk](https://github.com/gaearon/redux-thunk), the middleware that comes with react-redux-starter kit. However, we'll be keeping redux-thunk around for now. When you're working with Redux you need middleware to handle asynchronous actions, these are called side effects. That's why you see alternates to Redux Saga named things like [redux-effects](https://github.com/redux-effects/redux-effects) and [redux-side-effects](https://github.com/gregwebs/redux-side-effect). Redux Thunk works well for side effects but the other effects libraries make it [much easier to chain your effects](http://stackoverflow.com/questions/32925837/how-to-handle-complex-side-effects-in-redux). The biggest advantage of Redux Saga is that it uses native generator functions that are perfectly suited to the types of complex chained actions that you need for making asynchronous requests.

### Add sagas to createStore
We want to reuse the clever way that the react-redux-starter-kit loads the reducers for a route. You can read about it in detail [in the previous note](react-redux-starter-kit-todos.md#route-srcroutestodosindexjs). At a high level, react-redux-starter-kit provides an example of using Webpack to break routes into dynamic loading chunks. Part of that functionality involves loading the reducers for that route. This is helpful when you're using the [fractal project structure](https://github.com/davezuko/react-redux-starter-kit/wiki/Fractal-Project-Structure).

Thankfully redux-saga has [support for dynamically loading sagas](https://github.com/yelouafi/redux-saga/releases/tag/v0.1.0) in a similar fashion to the `injectReducer()` method that comes with react-redux-starter-kit. You may like to read about [how dynamically loading reducers works](https://github.com/reactjs/redux/issues/37). There is an issue that describes [the need for dynamically loading sagas](https://github.com/yelouafi/redux-saga/issues/76). The basics look like this:

We need to import our rootSaga and the sagaMiddleware from our `sagas.js` file (we'll make that file next). We also need to run our middleware with the root saga before returning the store. Redux-saga requires you to "run" your sagas. It's not important how it works but you can't leave this step off.

**`src/store/createStore.js`** (compare to [starter-kit version](https://github.com/davezuko/react-redux-starter-kit/blob/master/src/store/createStore.js))

```js
// ...

import reducers from './reducers'
import rootSaga, { sagaMiddleware } from './sagas'

export default (initialState = {}, history) => {
  let middleware = applyMiddleware(thunk, routerMiddleware(history), sagaMiddleware)

  // ...

  sagaMiddleware.run(rootSaga)

  return store
}
```

#### Don't run an empty root saga
If you are not running any sagas app-wide then you can shorten the code above to simply apply the middleware. You might do this if your `src/store/sagas.js` file (see below) returns an empty saga. That might likely be the case if you're dynamically loading your sagas in your route.

**`src/store/createStore.js`** (without app-wide sagas)

```js
// ...

import reducers from './reducers'
import { sagaMiddleware } from './sagas'

export default (initialState = {}, history) => {
  let middleware = applyMiddleware(thunk, routerMiddleware(history), sagaMiddleware)

  // ...

  return store
}
```


### Add root sagas
We need to create our `sagas.js` file to provide similar functionality to the `reducers.js` file. We need to add an `injectSaga()` function that works very similarly to `injectReducer()`. Because of how redux-saga manages tasks we need to provide a `name` for our saga. Within redux-saga, the [`sagaMiddleware.run(saga)`](http://yelouafi.github.io/redux-saga/docs/api/index.html#middlewarerunsaga-args) function returns a [`task`](http://yelouafi.github.io/redux-saga/docs/api/index.html#task-descriptor). This is important because redux-saga simply runs your tasks. If you're not careful you might accidentally start the same saga twice. If you find yourself with double execution bugs, then you're probably running the same saga more than once. Thankfully we replicated similar logic to `injectReducer()` and calling `injectSaga()` twice with the same arguments has no effect. And if you call it twice with the same name and a different saga, then it will cancel the previous saga and run the new one.

We also need a `cancelTask(name)` function for cancelling our named tasks. This makes it possible to manage sagas dynamically from our routes. We can effectively run the saga on enter and cancel the saga on leave.

If you are using app-wide sagas, these are saga that aren't related to a specific route and should be running at all times, then you should import them in this file and include them in the array returned by the `rootSaga`.

**`src/store/sagas.js`**

```js
import createSagaMiddleware from 'redux-saga'

// import sync sagas here
// import { helloSaga, watchIncrementAsync } from '../redux/modules/example'

export const sagaMiddleware = createSagaMiddleware()

// use injectSaga() to import from a route
const tasks = {}
export const injectSaga = ({ name, saga }) => {
  let { task, prevSaga } = tasks[name] || {}

  if (task && prevSaga !== saga) {
    cancelTask(name)
    task = undefined
  }

  if (!task || !task.isRunning()) {
    tasks[name] = {
      task: sagaMiddleware.run(saga),
      prevSaga: saga
    }
  }
}

export const cancelTask = (name) => {
  const { task } = tasks[name]
  tasks[name] = undefined

  if (task) {
    task.cancel()
  }
}

export default function * rootSaga () {
  yield [
    // Add sync sagas here
    // helloSaga(),
    // watchIncrementAsync()
  ]
}
```

## Dynamically load a saga from a route
We need to dynamically load our saga in our route. 

**`src/routes/Todos/index.js`**

```js
import { injectReducer } from '../../store/reducers'
import { injectSaga, cancelTask } from '../../store/sagas'

export default (store) => ({
  path: 'todos',
  getComponent (nextState, cb) {
    require.ensure([], (require) => {
      const TodosView = require('./components/TodosView').default
      const module = require('./modules/todos')
      const reducer = module.default
      const saga = module.rootSaga

      injectReducer(store, { key: 'todos', reducer })
      injectSaga({ name: 'todos', saga })

      cb(null, TodosView)
    }, 'todos')
  },
  onLeave () {
    cancelTask('todos')
  }
})
```

## Add sagas to a module

**`src/routes/Todos/modules/todos.js`**

```js

```
