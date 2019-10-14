# Vue3.0 尝鲜

国庆期间Vue3.0的源码出世，借着国庆归来划水一周的时间了解了3.0的一些主要新功能与源码的探索，在此做个小总结～

## 怎样入手看源码
将项目克隆到本地`git clone git@github.com:vuejs/vue-next.git`，首先注意到readme最后写的[Contributing Guide](https://github.com/vuejs/vue-next/blob/master/.github/contributing.md)，果断点进去看到`Development Setup` 部分，这里会有项目的开发设置介绍，用`yarn`命令安装相关依赖之后，在看package.json的scripts脚本命令都有哪些，因为项目刚出来，源码还是很清晰的，Contributing Guide中也有介绍这些脚本的作用，其中整个项目涉及到的相关工具：
- 整个项目完全基于TypeScript开发
- Rollup作为打包工具
- jest用来单元测试
- lerna是monorepo，进行多仓库管理

对这些开发工具🔧有一些了解，对看源码可以起到事半功倍的效果，都不了解的话对看源码也没多大问题

在大致看下项目目录结构，scripts目录下主要放了相关打包命令走的程序，packages目录是各个独立的核心源码包，其中的各个包依赖通过 [lerna](https://github.com/lerna/lerna) 管理，Contributing Guide中可以看到scripts脚本命令的含义，我们在命令行执行`yarn dev`，从控制台打印的信息
```
bundles packages/vue/src/index.ts → packages/vue/dist/vue.global.js...
```
也可以看出这个命令默认是以packages/vue/src/index.ts下的文件为打包入口，并打包至packages/vue/dist/下的vue.global.js文件下，注意文件名中：
- `global`: 表示打包的文件可直接在`<script>`中的src属性直接引入
- `esm`: 是在依赖于打包器的模块中使用import
- `esm-browser`: 在浏览器的ES模块中使用`<script type="module">`
- `cjs`: nodejs中通过`require()`使用

执行`yarn dev`命令后即可在packages/vue/dist/目录下看到vue.global.js文件，这个文件可以直接在`<script>`脚本中引入，这样就方便了我们去调试源码。

关于调试源码的方式，我个人很倾向于直接在vscode中进行调试，可以看我写的另一篇超详细讲解怎样在vscode中调试源码的文章，[在这里](https://github.com/Cslove/Blog/blob/master/learn-debugging-in-vscode.md)

回到这里，我们可以直接在项目的根目录下创建一个index.html文件并填入以下内容，
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
</head>
<body>
    <div id="app"></div>
    <script src="packages/vue/dist/vue.global.js"></script>
    <script>
        console.log(Vue)
    </script>
</body>
</html>
```
通过`<script>`标签直接引入打包出来的文件，然后在另一个script标签内直接打印Vue全局变量，将此文件从浏览器打开在控制台就可以看到打印信息

通过上面控制台打印的`Vue`信息，可以看到`Vue`已经不在是个`constructor`，也就是说你不能在这样使用`new Vue(options)`，它就像react一样暴露了众多api，具体可以看这个 [rfc](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0004-global-api-treeshaking.md) 

> *以下内容只针对Vue3.0的新功能探索，其中的一些新api最好看下此 [Composition API RFC](https://vue-composition-api-rfc.netlify.com/#summary)*

在上面的index.html文件下的第二个`<script>`块简单写如下代码：
```js
const {
    reactive,
    computed,
    watch,
} = Vue
var app = Vue.createApp().mount({
    setup() {
        const state = reactive({
            count: 0,
            double: computed(() => state.count * 2)
        })
        function increment() {
            state.count++
        }
        watch(() => {
            console.log(state.double)
        })
        return {
            state,
            increment,
        }
    },
    template: `<div>{{ state.count }}|{{ state.double }}</div>
    <button @click="increment">+ 1</button>`,
}, '#app')
```
保存文件后，刷新浏览器可看到就是一个小按钮点击后会有+1的小功能效果。在看上面的代码是与之前的写法完全不同的形式，这就是官方称为Composition的api，我觉得它主要代表了这几个目的：

- 逻辑复用
- 代码重组织
- 更好的类型接口

之前的Vue写法会要在各个生命周期不停的跳转查看逻辑，随着业务需求的增多，逻辑代码也越来越分散，这也导致了越来越难以复用和维护的问题。而且之前的版本在越来越大项目上typescript的支持度也不是很高，由于Vue本质在this的设计问题上，this是指向组件实例上的，而且methods下的函数中的this也是指向组件实例，就导致类型设计的复杂度很高，Vue 3.0就是解决这些问题的。

熟悉react的人看到上面的代码可能会说，这不就是react hooks嘛，是的，确实借鉴了react hook 相同的逻辑复用目的。但与hook又有重大的不同：

- `setup()`函数只会调用一次，它在`beforeCreate`之后，`created`hook之前调用
- react hooks会很在意调用顺序并且不能在条件语句中调用，而在`setup`中的Composition API不在乎
- Vue会自动追踪数据的更新，不像useEffect那样更新需要传入要依赖的数据数组

显然，无论怎样不同，Vue还是受到了react hook灵感而来

在回到上面的代码，与之前的形式不同，现在我们需要调用`createApp()`方法创建一个app实例，然后调用实例的`mount(rootComponent, rootContainer, rootProps)`方法，其接受三个参数，第一个是与之前的`new Vue(options)`中的options一样的根组件实例对象，第二个参数是应用要挂载的dom节点，第三个参数是组件的默认props

`setup()`是组件的一个新属性，它主要用于执行所有的Composition API的入口，是的，所有逻辑都可以写到这个函数里，Vue也相应的暴露出了所有的生命周期钩子，`onMounted()、onUpdated()、onXXX()...`这可能会导致setup函数会很膨胀，但其实这些逻辑就可以专门提取出来以供其他地方复用，这也就是Composition逻辑复用的目的

setup函数内部使用`reactive()`方法初始化了一个state对象，经过`reactive()`方法会返回一个被Proxy代理过的对象（后面会讨论内部的实现原理），`computed()`方法类似与原options方式的使用，但又有些微不同，computed方法实际返回的是个refs对象，类似与react的useRef()，`{ value: xx }`，需要拿到里面的value值才能拿到最终的值：
```js
const double = computed(() => state.count)

watch(() => {
    console.log(double.value) // 0
    console.log(double) // 一个ref对象
})
```
值得注意的是在`reactive()`方法内部和在模版里拿到computed之后的值的话，内部会自动拿到value值供使用，例如：
```js
const state = reactive({
    count: 0,
    double: computed(() => state.count * 2)
})

watch(() => {
    console.log(state.double) // 0，无需在这样获取： state.double.value
})
```
为什么要这样设计？主要原因还是js中原始类型和引用类型的概念，在`computed()`内部源码中能看到类似如下形式的代码：
```js
function computed(getter) {
  let value
  watch(() => {
    value = getter()
  })
  return value
}
```
这样的形式要是基本类型的值会不起作用的，因为值传递的是一个副本，要是改为如下：
```js
function computed(getter) {
  const ref = {
    value: null
  }
  watch(() => {
    ref.value = getter()
  })
  return ref
}
```
这样总是将`getter()`值付给一个ref对象的value值，我们拿到始终是一个引用类型的值，要复出的代价就是我们始终都要在外面手工获取它的value值，要是逻辑复杂的情况下，这样确实也会多一层心智负担。正因如此，Vue3专门引进了refs概念，可将它与react的`useRef()`hook概念做比较，还专门为我们提供了`ref()、isRef()、toRefs()`api，详情查看[API](https://vue-composition-api-rfc.netlify.com/api.html)

回到`setup()`函数，它可以返回一个渲染函数，或是一个对象，如果是个对象的话，对象内的每个属性会作为作践模版的渲染上下文使用，也就是与之前的`data()`函数返回的对象一样可在组件模版中使用，而且`setup()`函数返回的对象直接与this进行了绑定，可以直接在原来的组件options用this使用，方便与vue2.x进行组合使用。

