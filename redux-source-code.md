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
    [$$observable]: observable,
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

