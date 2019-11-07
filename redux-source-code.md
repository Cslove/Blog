# Redux 源码剖析

Redux本身只暴露了5个API，分别是：
```js
export {
  createStore,
  combineReducers,
  bindActionCreators,
  applyMiddleware,
  compose,
  __DO_NOT_USE__ActionTypes  // redux内部使用的action-type，可忽略
}
```
这篇文章将每个api的职责一一做个剖析总结，redux本身可用一句话概括：

> 可预测的状态管理容器

## createStore

createStore是redux核心需要调用的，返回一个store对象，包含针对全局状态树的“增删改查”操作，它主要有两种调用方式：`createStore(reducer, preloadState?)`和`createStore(reducer, preloadState?, enhancer?)`

- `reducer(state, action)`: 根据指定action返回一个新的state， 更改状态的核心逻辑所在
- `preloadState`: 初始全局状态（object）
- `enhancer`: 一个函数，返回一个加强版的store(主要针对dispatch函数)

后面两个参数是可选参数，createStore调用后返回一个store对象：

```js
store = {
    dispatch,
    subscribe,
    getState,
    replaceReducer,
}
```

接下来我们深入createStore函数内部，如果enhancer第三个参数存在并是一个函数，或者调用createStore只有两个参数且第二个参数是个函数，则会有以下逻辑：

```js
if (typeof enhancer !== 'undefined') {
    if (typeof enhancer !== 'function') {
      throw new Error('Expected the enhancer to be a function.')
    }
    return enhancer(createStore)(reducer, preloadedStat)
}
```
可以看到直接调用enhancer并返回了它，返回的就是一个与上面结构一致的加强版store对象，具体enhancer的实现我们在讲到`applyMiddleware`API的时候在谈。如果enhancer不存在的话，就会执行以下逻辑（简单起见我写了个简易版的实现，其他具体会在后面一起讲解源码的设计细节）：
```js
function createStore(reducer, preloadState) {
    let state = preloadState
    let listeners = []
    function getState() {
        return state
    }
    function subscribe(listener) {
        listeners.push(listener)
        return function unsubscribe() {
            var index = listeners.indexOf(listener)
            listeners.splice(index, 1)
        }
    }
    function dispatch(action) {
        state = reducer(state, action)
        listeners.forEach(listener => listener())
        return action
    }
    dispatch({})
    return { dispatch, subscribe, getState }
}
```

- `getState()`: 返回当前的全局状态对象
- `subscribe(listener)`: 订阅一个监听器（函数）
- `dispatch(action)`: 根据指定的action执行对应的reducer逻辑并返回最新的state，然后执行订阅的所有监听器，也是更改全局状态的唯一方法

在listener回调函数中需要调用`store.getState()`拿到最新的state，在执行其他逻辑，关于为什么不把state当成参数直接给每个listener回调，可以看看这个[FAQ](https://redux.js.org/faq/designdecisions#why-doesnt-redux-pass-the-state-and-action-to-subscribers)，上面就是redux的最原始简单实现，大家应该都能看懂，但肯定上不了生产的，有很多注意点需要专门提出来说下

首先在reducer中可以再次进行dispatch调用吗？源码设计中是不可以的，显然如果可以在reducer中再次执行dispatch操作，对性能是一个很大的隐患，会多次对listeners中的监听器进行多次渲染，因此社区也有很多插件其实也是可以进行批量dispatch的，例如`redux-batch`, `redux-batched-actions`，这些都是上面提到的提供了enhancer函数也就是一个加强版的dispatch函数，后续会提到enhancer。因此，redux的dispatch方法源码中有以下逻辑：
```js
const isDispatching = false
function dispatch(action) {
    // ...
    if (isDispatching) {
        throw new Error('Reducers may not dispatch actions.')
    }

    try {
        isDispatching = true
        state = reducer(state, action)
    } finally {
        isDispatching = false
    }
    // ...
}
```
值得一提的是isDispatching变量也在`subscribe`和`unsubscribe`中使用了，也就是说，在reducer中也是不能进行`store.subscribe()`和取消订阅操作的。

在想一下，可以在listener监听函数中再次执行`store.subscribe()`吗？想一下应该是可以的，可是看下我们上面的简易实现，如果在forEach循环的listener中再次执行`listeners.push(listener)`或者调用相应的unsubscribe函数可能会导致bug，因为push和splice操作都是改变了原数组。显然，这里需要两个listeners来防止这种情况出现，在源码中声明了两个变量以及一个函数：
```js
let currentListeners = []
let nextListeners = currentListeners

// 拷贝一份currentListeners到nextListeners
function ensureCanMutateNextListeners() {
    if (nextListeners === currentListeners) {
      nextListeners = currentListeners.slice()
    }
}
```
然后在subscribe函数体中，以及返回的unsubscribe中：
```js
function subscribe(listener) {
    ensureCanMutateNextListeners()
    nextListeners.push(listener)

    return function unsubscribe() {
        ensureCanMutateNextListeners()
        const index = nextListeners.indexOf(listener)
        nextListeners.splice(index, 1)
        currentListeners = null
    }
}
```
这里订阅和取消订阅操作都是在nextListeners上进行的，那么dispatch中就肯定需要在currentListeners中进行循环操作：
```js
function dispatch(action) {
    // ...
    const listeners = (currentListeners = nextListeners)
    for (let i = 0; i < listeners.length; i++) {
      const listener = listeners[i]
      listener()
    }
    // ...
}
```
如此设计就会避免相应的bug，但这样有一个明显的点要记着的就是，每次在listener回调执行订阅操作的一个新listener不会在此次正在进行的dispatch中调用，它只会在下一次dispatch中调用，可以看作是在dispatch执行前的listeners中已经打了个快照了，这次的dispach调用中在listener回调中新增的listener只能在下个dispatch中调用，因为currentListeners里还没有最新的listener呢

可以看到store中还返回了一个replaceReducer方法，直接粘贴源码如下：
```js
function replaceReducer(nextReducer) {
    if (typeof nextReducer !== 'function') {
      throw new Error('Expected the nextReducer to be a function.')
    }
    currentReducer = nextReducer

    dispatch({ type: ActionTypes.REPLACE })
    return store
}
```
replaceReducer方法本身的逻辑就如此简单，正如字面意思就是替换一个新的reducer，`dispatch({ type: ActionTypes.REPLACE })`这行与上面的简版代码`dispatch({})`效果类似，每当调用replaceReducer函数时都会以新的ruducer初始化旧的state并产生一个新的state，它比较常用于动态替换reducer或者实现热加载时候使用

## compose

compose函数在redux中的职责更像是一个工具函数，可以把它想象成一个组合器，专门将多个函数组合成一个函数调用，源码也很简单，如下：
```js
function compose(...funcs) {
  if (funcs.length === 0) {
    return arg => arg
  }
  if (funcs.length === 1) {
    return funcs[0]
  }
  return funcs.reduce((a, b) => (...args) => a(b(...args)))
}
```
它主要是利用了数组的reduce方法将多个函数汇聚成一个函数，可将它看作成这个操作：`compose(a, b ,c) = (...arg) => a(b(c(...arg)))`，从这个形式可以看出传给compose函数的参数必须都是接受一个参数的函数，除了最右边的函数（即以上的c函数）可以接受多个参数，它暴露出来成一个独立的api多用于组合enhancer

## applyMiddleware

applyMiddleware函数可以说是redux内部的一个enhancer实现，它可以出现在createStore方法的第二个参数或第三个参数调用`createStore(reducer, preloadState, applyMiddleware(...middlewares))`，redux相关插件基本都要经过它之口。它接受一系列的redux中间件`applyMiddleware(...middlewares)`并返回一个enhancer，我们先来看下它的源码：
```js
function applyMiddleware(...middlewares) {
  return createStore => (reducer, ...args) => {
    const store = createStore(reducer, ...args)
    let dispatch = () => {
      throw new Error(
        'Dispatching while constructing your middleware is not allowed. ' +
          'Other middleware would not be applied to this dispatch.'
      )
    }
    const middlewareAPI = {
      getState: store.getState,
      dispatch: (action, ...args) => dispatch(action, ...args)
    }
    const chain = middlewares.map(middleware => middleware(middlewareAPI))
    dispatch = compose(...chain)(store.dispatch)

    return {
      ...store,
      dispatch
    }
  }
}
```
以上就是applyMiddleware的所有源码，看redux源码的所有api设计就有个感觉，看起来都很精简。我们从它的参数看起，它接受一系列的middleware，redux官方说middleware的设计应遵循这样的签名`({ getState, dispatch }) => next => action`，为什么要这样设计呢？我们一步一步来看，先看下它的返回值，它返回的是一个函数，类似这样的签名`createStore => createStore`，从以上代码来看类似这种`createStore => (reducer, ...args) => store`，所以这也常常被用作createStore的第三个参数存在，还记得我们在讲createStore函数时说了，若是enhancer存在会有如下逻辑：
```js
if (typeof enhancer !== 'undefined') {
  if (typeof enhancer !== 'function') {
    throw new Error('Expected the enhancer to be a function.')
  }
  return enhancer(createStore)(reducer, preloadedStat)
}
```
刚好对应于我们上面写的函数签名，也就是说如果enhancer不存在，redux会创建内部的store，如果存在，就先创建自己内部的store，然后将store传给中间件进行“魔改”，“魔改“什么呢？`dispatch`函数，看一下它最后的返回值`return { ...store, dispatch }`覆盖了原生的dispatch方法，但并不是说原生的dispatch方法不见了，它只是经过中间件而被加工赋予了更多的功能，接着往下看最核心的两行
```js
const chain = middlewares.map(middleware => middleware(middlewareAPI))
dispatch = compose(...chain)(store.dispatch)
```
从第一行很容易可以看出它执行了众多中间件，以getState和dispatch方法为命名参数传递给中间件，对应于我们上面说的这样的签名`({ getState, dispatch }) => xxx`，这样上面的chain常量就应该是这个样子`[next => action => {}, next => action => {}, xxx]`，最关键的就是上面第二行代码了，compose函数我们已经解释过了，假设chain常量是这样`[a, b]`，那么就会有如下代码：
```js
dispatch = ((...args) => a(b(...args)))(store.dispatch)
```
看了上面的代码，可能有的人觉得看着更复杂了，其实`compose(...chain) = (...args) => a(b(...args))`，然后就是个立即执行函数
```js
(a => {
  console.log(a)  // 立即打印 1
})(1)
```
然后从上面我们就可以得出这块代码：`dispatch = a(b(store.dispatch))`这里我们就要去解释为什么middleware要设计成这样`({ getState, dispatch }) => next => action`，我们在进行一步一步拆解如下:
```js
b(store.dispatch) = action => {
  // store.dispatch可在此作用域使用，即 next
}
```
而`action => {}`不就是redux中dispatch的函数签名嘛，所以`b(store.dispatch)`就被当成一个新的dispatch传递给`a()`，a在以同种方式循环下去最终赋给dispatch，值得注意的是每个中间件最终返回值应这样写`return next(action)`，最终
```js
dispatch = action => next(action)
```
action变量信息就会随着next一步步从a到b穿透到最后一个中间件直至被redux内部的store.dispatch调用，也就是最终修改reducer逻辑，store.dispatch最终返回的还是action，这就是中间件逻辑，你可以在中间件中任何时候调用next(action)，最终返回它就行了，我们可以看个redux官网的小例子：
```js
function logger({ getState }) {
  return next => action => {
    console.log('will dispatch', action)
    // Call the next dispatch method in the middleware chain.
    const returnValue = next(action)
    console.log('state after dispatch', getState())

    // This will likely be the action itself, unless
    // a middleware further in chain changed it.
    return returnValue
  }
}

const store = createStore(todos, ['Use Redux'], applyMiddleware(logger))

store.dispatch({
  type: 'ADD_TODO',
  text: 'Understand the middleware'
})
// (These lines will be logged by the middleware:)
// will dispatch: { type: 'ADD_TODO', text: 'Understand the middleware' }
// state after dispatch: [ 'Use Redux', 'Understand the middleware' ]
```
