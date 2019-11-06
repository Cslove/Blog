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
    var state = preloadState
    var listeners = []
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
如此设计就会避免相应的bug，但这样有一个点要记着的就是，每次在listener回调执行订阅操作的一个新listener不会在此次正在进行的dispatch中调用，它只会在下一次dispatch中调用，可以看作是在dispatch执行前的listeners中已经打了个快照了，这次的dispach调用中在listener回调中新增的listener只能在下个dispatch中调用

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