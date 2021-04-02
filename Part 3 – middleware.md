## Intro

In Part 2, we covered `createStore`, which believe it or not, is the bulk of how Redux works. That being said, there is another crucial aspect of Redux and that is how it can be extended with custom functionality through the use of `applyMiddleware(...middleware)`.



According to the official API docs, the idea of `applyMiddleware(...middleware)` is to provide a single standard way extend `dispatch` in the ecosystem. This way, different middleware may compete in expressiveness and utility, which makes it easier for developers to find the suitable ones to use. 



Middleware lets us wrap the store's `dispatch` method so that we can extend it with custom functionality. Very importantly, middleware is composable, which means that multiple middleware can be combined together, where each middleware requires no knowledge of what comes before or after it in the middleware chain.



By reading the source code of `applyMiddleware`, I hope to address the following two questions

1. How exactly does the `dispatch` method get wrapped？
2. How does the next middleware in the middleware chain wrap the previous middleware's wrapped `dispatch` method (if that's the case)?



To answer these questions, let's first keep reading the API docs.



## API

`applyMiddleware` takes an array of middleware, which are functions that conform to the Redux middleware API. That is to say that they all have the function signature

```typescript
({ getState, dispatch }) => next => action
```



In practice, we can use object destructuring to use only one of `getState` and `dispatch`. For instance:

```javascript
function logger({ getState }) {
  return next => action => {
    // ...
  }
}
```



The return value of `applyMiddleware` is a store enhancer that's used by `createStore` to return an enhanced `createStore`, which has the custom extended functionality that we desire. 



Recall, how `createStore` takes a reducer and optionally takes a `preloadedState` and/or an `enhancer`. Putting it all together, we have:

```javascript
const store = createStore(
	reducer,
  applyMiddleware(...middleware)
)
```



With this high level understanding of `applyMiddleware`, we can now dive into the source code.

## Source Code

### `applyMiddleware` Signature

Again, let's first start with the function signature. 

```typescript
export default function applyMiddleware(): StoreEnhancer
export default function applyMiddleware<Ext1, S>(
  middleware1: Middleware<Ext1, S, any>
): StoreEnhancer<{ dispatch: Ext1 }>
export default function applyMiddleware<Ext1, Ext2, S>(
  middleware1: Middleware<Ext1, S, any>,
  middleware2: Middleware<Ext2, S, any>
): StoreEnhancer<{ dispatch: Ext1 & Ext2 }>
export default function applyMiddleware<Ext1, Ext2, Ext3, S>(
  middleware1: Middleware<Ext1, S, any>,
  middleware2: Middleware<Ext2, S, any>,
  middleware3: Middleware<Ext3, S, any>
): StoreEnhancer<{ dispatch: Ext1 & Ext2 & Ext3 }>
export default function applyMiddleware<Ext1, Ext2, Ext3, Ext4, S>(
  middleware1: Middleware<Ext1, S, any>,
  middleware2: Middleware<Ext2, S, any>,
  middleware3: Middleware<Ext3, S, any>,
  middleware4: Middleware<Ext4, S, any>
): StoreEnhancer<{ dispatch: Ext1 & Ext2 & Ext3 & Ext4 }>
export default function applyMiddleware<Ext1, Ext2, Ext3, Ext4, Ext5, S>(
  middleware1: Middleware<Ext1, S, any>,
  middleware2: Middleware<Ext2, S, any>,
  middleware3: Middleware<Ext3, S, any>,
  middleware4: Middleware<Ext4, S, any>,
  middleware5: Middleware<Ext5, S, any>
): StoreEnhancer<{ dispatch: Ext1 & Ext2 & Ext3 & Ext4 & Ext5 }>
export default function applyMiddleware<Ext, S = any>(
  ...middlewares: Middleware<any, S, any>[]
): StoreEnhancer<{ dispatch: Ext }>
export default function applyMiddleware(
  ...middlewares: Middleware[]
): StoreEnhancer<any>
```

In short, `applyMiddleware` takes some middleware and return a store enhacer. There are two types here that we need to look at, `StoreEnhacer` and `Middleware`.



**`StoreEnhancer`**

```typescript
export type StoreEnhancer<Ext = {}, StateExt = never> = (
  next: StoreEnhancerStoreCreator<Ext, StateExt>
) => StoreEnhancerStoreCreator<Ext, StateExt>
```

`StoreEnhacer` is a generic function that takes a function, `next`, which is of type `StoreEnhancerStoreCreator`:

```typescript
export type StoreEnhancerStoreCreator<Ext = {}, StateExt = never> = <
  S = any,
  A extends Action = AnyAction
>(
  reducer: Reducer<S, A>,
  preloadedState?: PreloadedState<S>
) => Store<ExtendState<S, StateExt>, A, StateExt, Ext> & Ex
```

`StoreEnhacerStoreCreator` takes a `reducer` and optionally a `preloadedState` and returns a `store` object. 



Aha! So, `StoreEnhacer` really just takes a store creator and returns a (new) store creator, which can be used to create a store object.



**`MiddleWare`**

```typescript
export interface Middleware<
  _DispatchExt = {},
  S = any,
  D extends Dispatch = Dispatch
> {
  (api: MiddlewareAPI<D, S>): (
    next: D
  ) => (action: D extends Dispatch<infer A> ? A : never) => any
}
```



We know that on a high level, a middleware is a function that conforms to the middleware API. The definition here exactly expresses this point. Although, TypeScript does get quite complicated when it tries to express a developer's intent precisely. Let's try to break it down.



We know that an interface specifies the structure of an object. Additionally, interfaces can be used to describe [function types](https://www.typescriptlang.org/docs/handbook/interfaces.html#function-types). Here, `Middleware` interface specifies a function that has the function signature (with a few details omitted):

```typescript
(api: MiddlewareAPI) => (next: D) => (action: D) => any
```



As for the whole `action: D extends Dispatch<infer A> ? A : never`, it is the TypeScript support for conditional typing. Semantically, this is saying that if `D` extends `Dispatch<infer A>`, then `action` is of type `A`; otherwise, `action` is of type `never`. Given the meaning of `never` in TypeScript, this is saying that `D` has to extend `Dispatch<infer A>`. 



As for `MiddlewareAPI`, we have:

```typescript
export interface MiddlewareAPI<D extends Dispatch = Dispatch, S = any> {
  dispatch: D
  getState(): S
}
```

 So an object is a `MiddlewareAPI` when it has two methods `dispatch` and `getState`. 



I hope you now see how this exactly conveys the high level intention. If not, consider this contrived example:

```typescript
interface DemoAPI {
    toString(): string
}

interface Demo {
    (api: DemoAPI): (input:any) => string
}

const demoAPI: DemoAPI = {
    toString() {
        return "demo api"
    }
}

const demo: Demo = demoAPI => {
    return (input: any) => ""
}
```



### `applyMiddleware` Body

If we omit some implementation details, then structurally, we have: 

```typescript 
return (createStore:  StoreEnhancerStoreCreator) => <S, A extends AnyAction>(
	reducer: Reducer<S, A>,
  preloadedState: PreloadedState<S>
) => {
  // ...
  return {
    ...store,
    dispatch
  }
}
```

Again, on a high level, we know that an enhancer is supposed to be returned by `applyMiddleware`. As we learned above, an enhancer is really just a function that takes a `StoreEnhancerStoreCreator` and returns a `StoreEnhancerStoreCreator`. As we can clearly see here, the `createStore` function itself is also of type `StoreEnhancerStoreCreator`. A `StoreEnhancerStoreCreator` takes a reducer and optionally a `preloadedState` and returns a new `Store` object.



That's why an `enhancer` is used in the `createStore` like this:

```typescript
return enhancer(createStore)(
  reducer,
  preloadedState as PreloadedState<S>
) as Store<ExtendState<S, StateExt>, A, StateExt, Ext> & Ext
```



Now, as for the omitted implementation details:

```typescript
  const store = createStore(reducer, preloadedState)
  let dispatch: Dispatch = () => {
    throw new Error(
      'Dispatching while constructing your middleware is not allowed. ' +
      'Other middleware would not be applied to this dispatch.'
    )
  }

  const middlewareAPI: MiddlewareAPI = {
    getState: store.getState,
    dispatch: (action, ...args) => dispatch(action, ...args)
  }
  const chain = middlewares.map(middleware => middleware(middlewareAPI))
  dispatch = compose<typeof dispatch>(...chain)(store.dispatch)

	return {
		...store,
		dispatch
	}
```

1. The provided `createStore` argument is used to create a `store` object, which will later have its `dispatch` method modified/wrapped. The modified `store` object is the return value when the returned function is invoked with a `reducer` and optionally a `preloadedState` as arguments.

2.  A temporary `dispatch` method is created, which if invoked while the current `middleware` is being constructed will throw an error.

3. Then we use the `store`'s `getState` and `dispatch` methods to create a `middlewareAPI`, which will be used to invoke each `middleware`: `middleware(middlewareAPI)`, which again returns a function of interface `Dispatch`.

4. The `chain`  created by `middlewares. map( middleware => middleware( middlewareAPI))` is used to create the final `dispatch` method.

5. The final `dispatch` method is created using `compose<typeof dispatch>(...chain)(store.dispatch)`, where the `compose` function is defined in `compose.ts`.

   ```typescript
   export default function compose<R>(...funcs: Function[]): (...args: any[]) => R
   
   export default function compose(...funcs: Function[]) {
     if (funcs.length === 0) {
       // infer the argument type so it is usable in inference down the line
       return <T>(arg: T) => arg
     }
   
     if (funcs.length === 1) {
       return funcs[0]
     }
   
     return funcs.reduce((a, b) => (...args: any) => a(b(...args)))
   }
   ```



Just like function composition in math, where $(f \circ g)(x) = f(g(x))$, we can compose functions in programming languages as well. This `compose` function takes the a list of functions and combines them from right to left. Let's see what this means:

- Say, we have a list of three functions `[f,g,h]`

- `[f,g,h].reduce((a,b) => (...args: any) => a(b(...args)))`

  1. `(f,g) => (...args) => f(g(...args))`

  2. let's use `i` to denote the function `(...args) => f(g(...args))`, then we have `(i, h) => (...args) => i(h(...args))`, which when we plug `(...args) => f(g(...args))` back gives `(...args) => f(g(h(...args)))`
  3. As there are no more functions left, the final return value is the function `(...args) => f(g(h(...args)))`



```typescript
const chain = middlewares.map(middleware => middleware(middlewareAPI))
dispatch = compose<typeof dispatch>(...chain)(store.dispatch)
```

The composed function `compose<typeof dispatch>(...chain)` is invoked with the `store.dispatch` as an argument. It's really the last function in the `chain` that's provided the `store.dispatch` as input. It's return value is then used as input for the previous function in the `chain`, whose return value is used for the one before, and so on and so forth.  



One of the implications of this, as noted in the API docs, is that when we have multiple middlewares that we wish to apply, we should put the potentially asynchronous ones before synchronous ones. Since the invocation order is from right to left. For instance, 'redux-thunk' should go before 'redux-devtools'. Consider this from the offcial API doc:

```javascript
let middleware = [a, b]
if (process.env.NODE_ENV !== 'production') {
  const c = require('some-debug-middleware')
  const d = require('another-debug-middleware')
  middleware = [...middleware, c, d]
}

const store = createStore(
  reducer,
  preloadedState,
  applyMiddleware(...middleware)
)
```

 















































