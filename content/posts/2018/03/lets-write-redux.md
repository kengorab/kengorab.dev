---
title: "Let's Write Redux!"
date: 2018-03-05T20:07:56-05:00
tags: [redux, javascript, diy]
---

_This is a copy of a post, [originally written on Medium](https://medium.com/@ken.gorab/lets-write-redux-9f6cb511b3e4). I've since rehosted it here._

---

As a corollary to the famous sentiment “the best way to prove mastery over a topic is to be able to teach it to somebody else”, I’d like to add “the best way to prove mastery over a piece of software it to implement it yourself.” I don’t know whether or not that’s true, but I do often enjoy building my own versions of the tools/libraries that I use. Not in any serious production environment or anything, just for the sake of learning how they work.

Anyway, now that the preamble is done, let’s write [Redux](https://redux.js.org/)!

## What Is Redux?
According to the documentation, it’s this:
> Redux is a predictable state container for JavaScript apps.

I’m not going to spend too much time talking about what Redux is (that’s not the point of this) but there are a couple of key parts which I need to mention.
First is the **store**, which is what houses your application’s data; second is the **reducer function(s)** which allow(s) you to define the per-action rules by which your store can be changed; lastly there is the **dispatch function** which allows you to specify the action that has occurred. The **dispatch function** is the gateway to changing state in Redux.

With that said, let’s get started writing Redux with the simple example from Redux’s README.

## Step 1: A Very Basic Example

```js
function counter(state = 0, action) {
 switch (action.type) {
   case ‘INCREMENT’:
     return state + 1
   case ‘DECREMENT’:
     return state — 1
   default:
     return state
 }
}

const store = createStore(counter)

store.subscribe(() => console.log(store.getState()))

store.dispatch({ type: ‘INCREMENT’ }) // logs 1
store.dispatch({ type: ‘INCREMENT’ }) // logs 2
store.dispatch({ type: ‘DECREMENT’ }) // logs 1
```

Though this example is extremely basic, it lays out nearly all aspects of Redux: the **store** (created with the `createStore` function) which consists only of 1 number, the **reducer function**, and the **dispatch function**.

Here’s what a skeleton of the `createStore` function might look like:
```js
function createStore(reducer) {
  function subscribe(callback) {
    // TODO
  }
  
  function getState() {
    // TODO
  }
  
  function dispatch(action) {
    // TODO
  }
  
  return {
    subscribe,
    getState,
    dispatch
  }
}
```

If it helps to think in terms of object-oriented programming, this `createStore` function is kind of acting as a class, which would have methods `subscribe`, `getState`, and `dispatch`. You _can_ write classes in javascript now (since ES2015), but to keep parity with the existing Redux API, I’ll stick with the functional approach. (For more reading on classes vs factory functions in javascript, I recommend [this Medium post](https://medium.com/javascript-scene/javascript-factory-functions-vs-constructor-functions-vs-classes-2f22ceddf33e)).

Let’s tackle the easiest functions first. The `subscribe` function simply registers a callback which will be invoked on every state change; this one is super easy. The `getState` function should return the existing state of the store, which means we’ll need a variable to keep track of it (you can think of this as an _instance variable_ in OOP-terminology). So, we’re now at this:

```js
function createStore(reducer) {
  let callback
  let currentState
  
  function subscribe(cb) {
    callback = cb
  }
  
  function getState() {
    return currentState
  }
  
  function dispatch(action) {
    // TODO still
  }
  
  return {
    subscribe,
    getState,
    dispatch
  }
}
```

Nothing earth-shattering yet, I hope. The final piece of the puzzle is the `dispatch` function. This function receives an action as a parameter, and is the gateway to all state changes within Redux. An **action** is simply an object literal that has a `type` property on it (you can see that the type is the subject of the `switch` statement in the reducer function, which is a very common pattern in reducer functions). The job of `dispatch` is to call the reducer function (`counter` in the simple example above) with the current state and the action it received. The return value from that reducer should then become the current state. Finally, it should invoke the subscription callback registered in the `subscribe` function. If that sounds like a lot, don’t worry it’s really not.

```js
function createStore(reducer) {
  let callback
  let currentState
  
  function subscribe(cb) {
    callback = cb
  }
  
  function getState() {
    return currentState
  }
  
  function dispatch(action) {
    const newState = reducer(currentState, action)
    currentState = newState;
    if (callback) {
      callback()
    }
  }
  
  return {
    subscribe,
    getState,
    dispatch
  }
}
```
<div class="caption">Lines 14-18 are the new part</div>

See, I told you it wasn’t a lot. The `reducer` that we invoke here is the function that was passed as a parameter to the `createStore` function, and it represents the **top-level reducer** (in the simple example, the top-level reducer is just the `counter` function, but this will eventually get more complicated in later examples). Whatever the reducer function returns will become the new state.

And we’re done! Well, almost. There are a few subtle things about the reducer function which were kind of glossed over. Let’s pull up that code again:

```js
function counter(state = 0, action) {
 switch (action.type) {
   case ‘INCREMENT’:
     return state + 1
   case ‘DECREMENT’:
     return state — 1
   default:
     return state
 }
}
```

I think the single trickiest thing about reducer functions is the fact that Redux will call your reducer for _every_ action; this is why it’s important that your reducer functions return the state that they’re passed (which will be the `currentState`) for action types that you don’t care about. Relatedly, state initially starts off as `undefined`, which would result in unexpected `NaN`s later on if not for that default parameter value in the function declaration.

One thing Redux does which we do not yet do, is fire off some unique action once the store has finished initializing such that no reducers handle that action. This effectively builds out the initial state since the reducer function will return its default value. To ensure the uniqueness of that action name (so nobody can write bad code which depends on Redux’s internals) Redux appends a randomized string to the end of that action. This is the last step in our implementation of `createStore`:

```js
function createStore(reducer) {
  let callback
  let currentState
  
  function subscribe(cb) {
    callback = cb
  }
  
  function getState() {
    return currentState
  }
  
  function dispatch(action) {
    const newState = reducer(currentState, action)
    currentState = newState;
    if (callback) {
      callback()
    }
  }
  
  const INIT_ACTION = 
    'INIT_' + Math.random()
      .toString(36)
      .substring(7)
      .split('')
      .join('.')
  dispatch({ type: INIT_ACTION })
  
  return {
    subscribe,
    getState,
    dispatch
  }
}
```
<div class="caption">Note the call to dispatch on line 27</div>

Okay, now we’re done. [Here is a codepen](https://codepen.io/anon/pen/rJgMwL?editors=0012) of the simple example and our implementation of Redux to handle it. I think it’s time to step up the complexity a bit. Though really, it’s not that large a step.

## Step 2: Combining Reducers

In the simple example above, the state consisted of a single number, a counter. That was fine for a simple example, but let’s try to tackle a more complex state; let’s build the state for a Todo App (yeah, I know it’s a bit tired at this point, but it’s a good example of a structure which many people understand, and it’s [the example in the Redux documentation](https://redux.js.org/basics/reducers)).

```js
function visibilityFilter(state = 'SHOW_ALL', action) {
  switch (action.type) {
    case 'SET_VISIBILITY_FILTER':
      return action.filter
    default:
      return state
  }
}

function todos(state = [], action) {
  switch (action.type) {
    case 'ADD_TODO':
      return [
        ...state,
        {
          text: action.text,
          completed: false
        }
      ]
    case 'TOGGLE_TODO':
      return state.map((todo, index) => {
        if (index === action.index) {
          return { ...todo, completed: !todo.completed }
        }
        return todo
      })
    default:
      return state
  }
}

const topLevelReducer = combineReducers({
  visibilityFilter,
  todos
})

const store = createStore(topLevelReducer)
```

Here we have two reducer functions, `todos` and `visibilityFilter`, which are passed into `combineReducers` to create a single top-level reducer. This top-level reducer is passed into the `createStore` call, like we’ve seen before. Really the only difference is that `combineReducers` function, so let’s dive into what that’s supposed to do.

Judging by the documentation, the result of that `combineReducers` function is _itself_ a reducer function, which returns an object of the shape
```
{
  visibilityFilter: ???,
  todos: ???
}
```
where the `???`s are the return values of each of those separate reducer functions. At this point I’ve typed the word “function” so much it’s starting to not look like a word anymore; let’s write the code for `combineReducers`.

```js
function combineReducers(reducers) {
  return function(state = {}, action) {
    const nextState = {}

    Object.keys(reducers).forEach(key => {
      const previousStateForKey = state[key]
      const reducerForKey = reducers[key]
      
      const nextStateForKey = reducerForKey(previousStateForKey, action)
      nextState[key] = nextStateForKey
    })
    
    return nextState
  }
}
```

Nothing magical going on here; the `reducers` map / object literal which was passed to `combineReducers` outlines the shape of the state and defines which reducer function maps to which property on the state. The return value of `combineReducers` is a reducer function itself which builds an empty object literal to represent the next state, calls each of its reducer functions with their previous sub-section of the state and the current action, and then assembles them all back together. Once you understand how ordinary reducer functions work, a “combined” reducer function is really nothing special.

[Here is a codepen](https://codepen.io/anon/pen/LQKxXp?editors=0011) of the working Todo example, using our `combineReducers` and `createStore` functions, and here is our entire implementation of basic Redux, in 50 lines:

```js

function createStore(reducer) {
  let callback
  let currentState
  
  function subscribe(cb) {
    callback = cb
  }
  
  function getState() {
    return currentState
  }
  
  function dispatch(action) {
    const newState = reducer(currentState, action)
    currentState = newState;
    if (callback) {
      callback()
    }
  }
  
  const INIT_ACTION = 
    'INIT_' + Math.random()
      .toString(36)
      .substring(7)
      .split('')
      .join('.')
  dispatch({ type: INIT_ACTION })
  
  return {
    subscribe,
    getState,
    dispatch
  }
}

function combineReducers(reducers) {
  return function(state = {}, action) {
    const nextState = {}

    Object.keys(reducers).forEach(key => {
      const previousStateForKey = state[key]
      const reducerForKey = reducers[key]
      
      const nextStateForKey = reducerForKey(previousStateForKey, action)
      nextState[key] = nextStateForKey
    })
    
    return nextState
  }
}
```

## Conclusion

That’s really all there is to basic Redux. I always recommend to anyone interested in using Redux that they read through [the source](https://github.com/reactjs/redux/tree/master/src) because there really isn’t much to it. Sure, the actual library has logging, assertions about the shape of the state, null-checking, and other such things, but the core functionality of Redux really is quite simple; all it’s really doing is calling your functions in the right order and assembling them together in the shapes you describe.

That being said though, there are a couple of things that I left out. The `createStore` function also optionally accepts 2 additional parameters: `preloadedState` and `enhancer`. The first allows you to define an initial value for your entire state up-front, for defrosting some persisted/stored state, or for many other reasons. Implementing this one is really as simple as assigning `currentState` to that value at the start instead of having it be undefined.

The second parameter, `enhancer` has a bit more to it, but it’s still not overly complex. An enhancer is a function which you can provide in order to modify Redux slightly in order to achieve certain things. For example, the concept of “middleware” or “functions that get automatically called on every dispatch” is achieved via an enhancer. I definitely recommend you check out [the official Redux documentation](https://redux.js.org/advanced/middleware) for this, as it walks through exactly how and why middlewares are useful, and I recommend you check out the [`applyMiddleware`](https://github.com/reduxjs/redux/blob/master/src/applyMiddleware.ts) function in Redux’s code for an example of an enhancer function.

## To Be Continued?

Most practical uses of Redux include making HTTP requests (or some other kind of asynchronous actions) or using it alongside some view library (React, for example). If I feel like it, I might write up a “Let’s Write Redux-React” or “Let’s Write Redux-Thunk” post at some point. But I think I’ll leave it here for now.

Thank you for reading, and I hope that helps demystify Redux a bit.
