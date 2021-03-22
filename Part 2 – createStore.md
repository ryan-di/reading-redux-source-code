Welcome back to the second post of the Reading Redux Source Code series. As mentioned in [Part 1] of this series, this second post will be about `createStore()`.

This series, as you've probably guessed it, is about my journey in reading the Redux source code. You might be wondering why I'm writing it in this way, not starting with the overall architecture/structure, but instead with this particular function. 

The truth is that's exactly how I read the source code, starting from a function that I'm interested in and let it guide me uncover the app structure. This way, I learn something about the source code pretty quickly, which motivates me to keep digging. 

I must say that if I were to teach someone Redux or its source code in a limited amount of time, I probably wouldn't do it this way. It's lengthy and since we are basically "going with the wind" it's also hard to keep track of things. But I think it's still valid and in fact very important to document my learning journey honestly. Sometimes, the raw and the messy are just as valuable. 

Anyways, with that out the way, let's dive in. 

---

On a high level, we know that `createStore` creates a Redux `store` that holds the complete state of our app. We must give `createStore` a reducer. Optionally, we can provide `preloadedState` and/or an `enhancer`. The `preloadedState` will be used as the initial state and the `enhancer` is a function that can be used to enhance the `store` with third-party capabilities. 

The `store` object returned from `createStore` holds the state of our application. This `store` object has methods that we can use to get the current state, update the state, and attach observers that will be invoked when the state gets updated.

Structurally, `createStore` is a factory function. It declares the internal states and defines the methods for `store`,  put them together to create and return a `store` object.

You can take a look at the official [Redux API reference](https://redux.js.org/api/createstore) .It's very well-written. 

## `createStore` function signature

After forking, downloading, and opening up the Redux repo in VS Code, I found `createStore` in `/src/createStore.ts` by searching for it in the code editor.  Just take a look at this function signature!

```typescript
export default function createStore<
  S,
  A extends Action,
  Ext = {},
  StateExt = never
>(
  reducer: Reducer<S, A>,
  enhancer?: StoreEnhancer<Ext, StateExt>
): Store<ExtendState<S, StateExt>, A, StateExt, Ext> & Ext
export default function createStore<
  S,
  A extends Action,
  Ext = {},
  StateExt = never
>(
  reducer: Reducer<S, A>,
  preloadedState?: PreloadedState<S>,
  enhancer?: StoreEnhancer<Ext, StateExt>
): Store<ExtendState<S, StateExt>, A, StateExt, Ext> & Ext
export default function createStore<
  S,
  A extends Action,
  Ext = {},
  StateExt = never
>(
  reducer: Reducer<S, A>,
  preloadedState?: PreloadedState<S> | StoreEnhancer<Ext, StateExt>,
  enhancer?: StoreEnhancer<Ext, StateExt>
): Store<ExtendState<S, StateExt>, A, StateExt, Ext> & Ext
```

What the hell is going on here? For starters, Redux is now written completely in TypeScript. Comparing this signature to its JavaScript predecessor

```tsx
export default function createStore(reducer, preloadedState, enhancer)
```

things have really "escalated"!

If you're familiar with TypeScript, by looking at the first 30 or so lines of code in the function body of `createStore()`, you can already get a good sense of what the function signature is trying to achieve:

```typescript
if (
  (typeof preloadedState === 'function' && typeof enhancer === 'function') ||
  (typeof enhancer === 'function' && typeof arguments[3] === 'function')
) {
  throw new Error(
    'It looks like you are passing several store enhancers to ' +
    'createStore(). This is not supported. Instead, compose them ' +
    'together to a single function.'
  )
}

if (typeof preloadedState === 'function' && typeof enhancer === 'undefined') {
  enhancer = preloadedState as StoreEnhancer<Ext, StateExt>
  preloadedState = undefined
}

if (typeof enhancer !== 'undefined') {
  if (typeof enhancer !== 'function') {
    throw new Error('Expected the enhancer to be a function.')
  }

  return enhancer(createStore)(
    reducer,
    preloadedState as PreloadedState<S>
  ) as Store<ExtendState<S, StateExt>, A, StateExt, Ext> & Ext
}

if (typeof reducer !== 'function') {
  throw new Error('Expected the reducer to be a function.')
}
```

Take a deep breath and let's dissect it together if you're relatively new to TypeScript! In addition to all the type stuff that we clearly see, there are also two important concepts on display in the function signature:

- [generic functions](https://www.typescriptlang.org/docs/handbook/2/functions.html#generic-functions), (`createStore<S, A extends Action, Ext = {}, stateExt = never>`)

- [function overloading](https://www.typescriptlang.org/docs/handbook/2/functions.html#function-overloads) (the multiple `export default function createStore...`)

I recommend you take a look at the links provided above. When you've got a good understanding of generics and function overloading, then come back and continue the journey with me. 

---

### Related Types

The first step that I took, after learning more about generics and function overloading in TypeScript, was to look up the type definitions of all the types that appeared in the overwhelming looking signature. I was in particular very interested in the definition of `Store`. In VS Code for mac, we can go to the definitions by holding the command key and click on the function, class, type. It should be very similar for Windows users. 

Here are some of the more relevant ones (I've decided to ignore `StoreEnhancer` for now).

`Action`

```typescript
export interface Action<T = any> {
  type: T
}
```

An action is a plain object that has a property whose name is `type` and whose value could any type.

`AnyAction`

```typescript
export interface AnyAction extends Action {
  [extraProps: string]: any
}
```

The `AnyAction` interface extends `Action` and allows any additional properties to be added. These extra properties are the information that we attach to an action. 

`Reducer`

```typescript
export type Reducer<S = any, A extends Action = AnyAction> = (
	state: S | undefined,
	action: A
) => S
```

We can see from the definition above that a reducer is indeed a function that takes a state and an action and returns another state. 

`Store`:

```typescript
export interface Store<
  S = any,
  A extends Action = AnyAction,
  StateExt = never,
  Ext = {}
> {
  dispatch: Dispatch<A>

  getState(): S

  subscribe(listener: () => void): Unsubscribe

  replaceReducer<NewState, NewActions extends Action>(
    nextReducer: Reducer<NewState, NewActions>
  ): Store<ExtendState<NewState, StateExt>, NewActions, StateExt, Ext> & Ext

  [Symbol.observable](): Observable<S>
}
```

In TypeScript, the keyword `interface` is used to define the structure of an object. An interface is made up of property names and their respective types. Just like generic functions, we can also make a generic object that conforms to an interface.

Here we can see that by default, a `store` object can hold state of any type and has methods `dispatch`, `getState`, `subscribe`, and `replaceReducer`. Lastly, it conforms to the Observable proposal, which we can ignore. 

All these methods are very well named and we can easily guess what they are supposed to do. 

Lastly, for `dispatch`, we see that is needs to take an action and returns the action.

```typescript
export interface Dispatch<A extends Action = AnyAction> {
  <T extends A>(action: T, ...extraArgs: any[]): T
}
```

## `createStore` function body

Back to the first 30 lines or so code:

```typescript
if (
  (typeof preloadedState === 'function' && typeof enhancer === 'function') ||
  (typeof enhancer === 'function' && typeof arguments[3] === 'function')
) {
  throw new Error(
    'It looks like you are passing several store enhancers to ' +
    'createStore(). This is not supported. Instead, compose them ' +
    'together to a single function.'
  )
}

if (typeof preloadedState === 'function' && typeof enhancer === 'undefined') {
  enhancer = preloadedState as StoreEnhancer<Ext, StateExt>
  preloadedState = undefined
}

if (typeof enhancer !== 'undefined') {
  if (typeof enhancer !== 'function') {
    throw new Error('Expected the enhancer to be a function.')
  }

  return enhancer(createStore)(
    reducer,
    preloadedState as PreloadedState<S>
  ) as Store<ExtendState<S, StateExt>, A, StateExt, Ext> & Ext
}

if (typeof reducer !== 'function') {
  throw new Error('Expected the reducer to be a function.')
}
```

It's mostly argument validation. Recall the API:

```tsx
createStore(reducer, [preloadedState], [enhancer])
```

We can gather the following from the source code:

1. `reducer` must be provided and it must be a function

   ```typescript
   if (typeof reducer !== 'function') {
     throw new Error('Expected the reducer to be a function.')
   }
   ```

2. if `preloadedState` is provided but `enhancer` is not and that `preloadedState` is a function, then it should be interpreted as an `enhancer`

   ```typescript
   if (typeof preloadedState === 'function' && typeof enhancer === 'undefined') {
     enhancer = preloadedState as StoreEnhancer<Ext, StateExt>
     preloadedState = undefined
   }
   ```

3. `enhancer` must be a function

   ```typescript
   if (typeof enhancer !== 'undefined') {
     if (typeof enhancer !== 'function') {
       throw new Error('Expected the enhancer to be a function.')
     }
     // ...
   }
   ```

4. we cannot provide more than one `enhancer` functions 

   ```typescript
   if (
     (typeof preloadedState === 'function' && typeof enhancer === 'function') ||
     (typeof enhancer === 'function' && typeof arguments[3] === 'function')
   ) {
     throw new Error(
       'It looks like you are passing several store enhancers to ' +
       'createStore(). This is not supported. Instead, compose them ' +
       'together to a single function.'
     )
   }
   ```

Very interestingly, we have

```typescript
return enhancer(createStore)(
  reducer,
  preloadedState as PreloadedState<S>
) as Store<ExtendState<S, StateExt>, A, StateExt, Ext> & Ext
```

An "enhanced" `createStore` is created by `enhancer`. This enhanced `createStore` is then used to create our `store` object. This is how third-party functionalities can be added to Redux. We'll come back to this in greater details in a later post.

---

Let's move on. 

```typescript
let currentReducer = reducer
let currentState = preloadedState as S
let currentListeners: (() => void)[] | null = []
let nextListeners = currentListeners
let isDispatching = false
```

The above five lines declare the internal states of `createStore()`. 

1. As `store` has a method called `replaceReducer()` to replace the current reducer, `currentReducer`, which better indicates the intention, is used to reference the given `reducer`.
2. `currentState` is initialized with the given `preloadedState`
3. `currentListeners` is used to store the list all the functions that have subscribed to the `store` object (recall that `store` object has a `subscribe` method)
4. `nextListeners` also stores the list of listeners but, as we will see, is mutable. This mechanism is very important in the managing of listeners.
5. Finally, `isDispatching` is used a simple semaphore lock that ensures no other actions will be performed while an action is being dispatched. We will see how it achieves this shortly. 



```typescript
function ensureCanMutateNextListeners() {
  if (nextListeners === currentListeners) {
    nextListeners = currentListeners.slice()
  }
}
```

`ensureCanMutateNextListeners()` is a helper function that makes `nextListeners` the shallow copy of `currentListeners` if they point to the same array that stores all the listeners. The implication of this small function is huge as we'll soon see. 



```typescript
function getState(): S {
  if (isDispatching) {
    throw new Error(
      'You may not call store.getState() while the reducer is executing. ' +
      'The reducer has already received the state as an argument. ' +
      'Pass it down from the top reducer instead of reading it from the store.'
    )
  }

  return currentState as S
}
```

Voila, we see the lock `isDispatching` in action. So for `getState()`, if the `store` object is not in the middle of dispatching an action, will return the current state.



Next, we have the `subscribe` method. 

```typescript
function subscribe(listener: () => void) {
  if (typeof listener !== 'function') {
    throw new Error('Expected the listener to be a function.')
  }

  if (isDispatching) {
    throw new Error(
      'You may not call store.subscribe() while the reducer is executing. ' +
      'If you would like to be notified after the store has been updated, subscribe from a ' +
      'component and invoke store.getState() in the callback to access the latest state. ' +
      'See https://redux.js.org/api/store#subscribelistener for more details.'
    )
  }

  let isSubscribed = true

  ensureCanMutateNextListeners()
  nextListeners.push(listener)

  return function unsubscribe() {
		// ...
  }
}
```

We see that while the `store` object is busy dispatching an action, no more listeners can be added. When it is okay to add a new listener, we define the `isSubscribed` boolean and set it `true`, which will be used by the returned `unsubscribe` function to determine whether the given listener has been removed. 

We see `ensureCanMutateNextListeners()` in use for the first time. It is used to ensure `nextListeners` can be mutated so that the new `listener` can be added to it. `ensureCanMutateNextListeners()`, though lengthy, is very appropriately named. Though, this usage here does not necessarily explain why we have to go that extra mile. We will see how it really is needed when we come to `dispatch`.

As for the returned unsubscribe function, we have:

```typescript
return function unsubscribe() {
  if (!isSubscribed) {
    return
  }

  if (isDispatching) {
    throw new Error(
      'You may not unsubscribe from a store listener while the reducer is executing. ' +
      'See https://redux.js.org/api/store#subscribelistener for more details.'
    )
  }

  isSubscribed = false

  ensureCanMutateNextListeners()
  const index = nextListeners.indexOf(listener)
  nextListeners.splice(index, 1)
  currentListeners = null
}
```

We can see how the closed `isSubscribed` boolean is used and how the given listener is removed. All thanks to closure. 

