One of the resolutions that I have for 2021 is to study some open source projects and learn from their architecture and code. It was not long ago that I started my journey in React, shortly after that I learned Redux. Instantly, I was captivated by how simple yet powerful its design philosophy was. Given the moderate size and the well-written docs, I decided to dive into its [source code](https://github.com/reduxjs/redux).

Through this *Reading Redux Source Code* series, I wish to share with you what I’ve learnt reading the source code. Please do not hesitate to contact me if you find any errors. I’d really appreciate it if you could give me some truthful feedback!

In this first post, I’d like to provide a brief overview of Redux. If you are already familiar with Redux and are only interested in the actual source code walkthrough, please follow me and come back for the next post, where I talk about `createStore()`.

If you’re new to Redux or would like a refresher on Redux, please read on.

As web app’s get more complex, we unavoidably have more app states that we need to keep track of. However, this is difficult! when a view updates a model, it might trigger another model to update, which might then cause another view to update… On top of that, some of these changes are very likely asynchronous. Very quickly, our human brain will no longer be able to keep track of what’s actually going on. Losing control over the when, why, and how of the app state, by all means, should be avoided as it easily leads to code that’s hard to debug, extend, and maintain.

Redux is a lightweight, extendable, and fantastically designed open source library for managing the ever-changing app state. Much like how the three paradigms of programming (procedural, OO, and functional) help make programmers’ lives simpler, Redux is able to make state management simple (not necessarily easy) not by giving us more power but by restricting how we manage our state. This can be reflected in the three principles of Redux:

**1. Sing source of truth**

The (global) state of your application is stored in an object tree within a single store object. This store object provides methods through which we can retrieve and update the state.

**2. State is read-only**

The only way to change the state is to emit an action, which is simply an object describing what happened. This object that must have a type property describing the action and may carry extra information about the action that just happened.

**3. Changes are made with pure functions**

To specify how the state tree is transformed by actions, you write pure reducers, which are functions that take in a state, an action and return a (new) state without any side effects.

For a quick taste of Redux, consider the following example:

```javascript
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
    case 'COMPLETE_TODO':
      return state.map((todo, index) => {
        if (index === action.index) {
          return Object.assign({}, todo, {
            completed: true
          })
        }
        return todo
      })
    default:
      return state
  }
}

import { combineReducers, createStore } from 'redux'
const store = createStore(reducer)
```

With the `store` object, which is the return value of `createStore(reducer)`, we can its methods to interact with the state stored in `store`. 

```javascript
store.dispatch({
  type: 'ADD_TODO',
  text: 'This is a new todo.'
})
```

With this method call, effectively, the `todos` reducer will be called with the current state and the given action object as arguments. As the action type is `'ADD_TODO'`, a new todo item will be added using the text that was provided. Very importantly, a new array was created with `[...state, {}]`. This way, we do not mutate the `state` directly and the returned array will be set as the new `state`. We can clearly see the three guiding principles of Redux at work even in this small example.

Below are some tutorials and materials that you can use to quickly get started with Redux:

- https://github.com/pshrmn/notes/blob/master/redux/redux.md
- https://www.freecodecamp.org/learn/front-end-libraries/#redux
- https://github.com/reduxjs/redux/tree/master/docs/tutorials/essentials

I’ll see in you in the next one where we discuss `createStore()`.