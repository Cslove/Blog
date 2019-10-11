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
可以看到我们通过`<script>`标签直接引入打包出来的文件，然后在另一个script标签内直接打印Vue全局变量，将此文件从浏览器打开在控制台可看到打印信息
