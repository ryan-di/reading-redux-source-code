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



As for `MiddlewareAPI`, we have:

```typescript
export interface MiddlewareAPI<D extends Dispatch = Dispatch, S = any> {
  dispatch: D
  getState(): S
}
```

 So an object is of `MiddlewareAPI` when it has two methods `dispatch` and `getState`.



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











































