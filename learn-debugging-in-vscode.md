# Debugging in VSCode

描述了在VS Code中怎样充分利用其强大的debug功能来调试web项目和源码调试（TypeScript）
 ## 背景

之前在看一些github源码的时候看调试数据是非常非常吃力，不知道多少时间消耗在了console，rebuild，switch tab的这几个频率最高的步骤中，着实烦躁。首先回顾一下我自己之前在看一些库源码的过程，以rollup这个打包库为例，首先`git clone`下来，`npm install`后发现它也自动执行了打包过程，看了下`package.json`文件中的scripts字段，其中有prepare脚本命令，npm在执行`npm install`后就会自动执行这个script，这个命令又指向了`npm run build`，进而进行了自动打包过程。通常我都会在这个project下执行`npm link`然后重新重建个文件夹写自己的一些例子项目link到前面的project，这样看project下的源码或者console之后重新build，就能在例子项目实时能够调试到，其中执行`npm run watch`可以不用重新build。

这样真的很累！...需要开两个项目频繁切换，消耗时间，还一点都不灵活！下面就研究了下VSCode中强大的Debug功能，让我犹如坐上了火箭🚀，并在此做个总结。

## 初识Debug Panel（牛刀小试）
下面是vscode中的debug panel

 ![image](https://code.visualstudio.com/assets/docs/editor/debugging/debugging_hero.png)

 最左边一列的第四个tab，之前的我看到过无数遍这个小蜘蛛，却从来没有打开过...，相信我，它会让你打开新世界的大门。(`command+shife+D`快捷打开此面板)

 方便这节后面的演示，用VSCode打开一个空目录取名Hello，下面新建一个app.js，写个最简单的代码如下：
 ```js
 var hello = "Hello World"
 console.log(hello)
 ```
 然后点击小蜘蛛进入到debug面板，点击👇下面的小齿轮图标

  <div align=center><img width="50%" src="https://code.visualstudio.com/assets/docs/editor/debugging/launch-configuration.png"/></div>


> *如果你的项目根目录下有.vscode目录且下面有launch.json配置文件，则点击上面的小齿轮按钮会直接打开此 launch.json配置文件*

这时会出现如下，让你选择调试环境，VSCode内置了Node.js环境，可以在插件tab安装其他语言的扩展，VSCode支持各种语言的调试，eg：PHP，Python，go，C/C++...我们直接选择Node.js环境

  <div align=center><img width="50%" src="https://code.visualstudio.com/assets/docs/editor/debugging/debug-environments.png"/></div>

选择后会直接打开一个launch.json文件并有如下配置：
```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "type": "node",
            "request": "launch",
            "name": "Launch Program",
            "program": "${file}"
        }
    ]
}
```

按`command+shift+E`回到目录列表会看到多了一个.vscode目录，下面就是该launch.json文件，可以点击右下角的Add Configuration按钮，会弹出如下配置供选择，在这个配置文件里可以充分利用它的智能感知功能，我们选择`Node.js: Launch Program`

<div align=center><img width="50%" src="https://github.com/Cslove/Blog/raw/master/screenshots/launch.png"/></div>

它会自动帮我们生成该配置的常用项，我们将其中的`"program": "${file}"`改为`"program": "${workspaceRoot}/app.js"`，在这个配置有挺多的内置变量可以直接使用，`${file}`就是当前所活跃的文件，`${workspaceRoot}`表示当前项目的工作根目录，完整的替换变量的列表可参考[这里](https://code.visualstudio.com/docs/editor/variables-reference)

> *在上面的配置字段中有个`request`字段，有两个值可以选择：`"launch"`和 `"attach"`，它表示VS Code中核心的两种调试模式。当时搞清这两种模式区别的时候也是有点晕，这也是取决与你项目中的工作流是怎样的，假如你调试的是web项目，通常你会在浏览器中已经直接打开项目了，这个时候我们可以使用attach模式，正如字面的意思，将debugger附加到你已经运行到的web项目，而launch就像字面意思是直接由编辑器启动这个程序，比如启动一个服务端项目或者上面我们的小例子，这个启动模式编辑器会自动把debugger附加到这个程序中。launch config 就像是作为一种debug模式直接启动app，而attach config 就像是debugger连接到正在运行的app。下面我们在web项目调试一节会有这两种模式的直观印象*

在开始debug之前，我们先在app.js的第二行最左边打一个断点

 <div align=center><img width="50%" src="https://github.com/Cslove/Blog/raw/master/screenshots/break_point.png"/></div>

`configurations`里面的name字段会显示在左边Debug一栏最上面的下拉列表里，点击小齿轮左边的框就可以选择刚才添加的配置，对应与`configurations`里面的name字段，选择Launch Program，然后点击左边的小绿色三角启动debug(F5)，然后就可以看到程序暂停在刚才打的断点这行了

<div align=center><img width="70%" src="https://github.com/Cslove/Blog/raw/master/screenshots/debug.png"/></div>

可以看到最上面的工具栏就是所有的调试操作：

![image](https://code.visualstudio.com/assets/docs/editor/debugging/toolbar.png)

- 继续/暂停 <font color="orange">F5</font>
- 单步跳过 <font color="orange">F10</font>
- 单步调试 <font color="orange">F11</font>
- 单步跳出 <font color="orange">⇧F11</font>
- 重启 <font color="orange">⇧⌘F5</font>
- 停止 <font color="orange">⇧F5</font>

左边的调试栏也显示出了程序里运行的所有变量，可以看到commonjs的模块变量也显示出来了`__dirname,__filename,module,require...`global下面也有所有的全局变量，调用堆栈也显示了程序中函数的调用顺序，这样你就可以在任何你想调试的地方打个断点，这个地方的所有信息就全部暴露出来了。下面我们可以看看怎样来调试web项目

## 调试web项目

我们用create-react-app官方脚手架创建一个简单app项目，在终端运行如下命令：

```bash
npx create-react-app debug-react  # npm 版本5.2以上
cd debug-react
npm start
```
可以看到浏览器自动打开并运行了这个简单web项目，为了方便调试看到效果，我们将App.js代码稍加改动如下：

```js
//  App.js
// ...
function App(props) {
  const {
    link,
    text,
  } = props  // 这行打个断点
  return (
    <div className="App">
      <header className="App-header">
        <img src={logo} className="App-logo" alt="logo" />
        <p>
          Edit <code>src/App.js</code> and save to reload.
        </p>
        <a
          className="App-link"
          href={link}
          target="_blank"
          rel="noopener noreferrer"
        >
          {text}
        </a>
      </header>
    </div>
  );
}
// ...
```
相应的index.js改动如下： 

```js
// index.js
// ...
ReactDOM.render(
    <App
        link="https://reactjs.org"
        text="hello world"
    />,
    document.getElementById('root')
);
// ...
```
保存后浏览器自动刷新了，可以看到只是将渲染数据提取出来以props形式从父组件传入，接下来我们进行debug的配置

调试web项目需要安装一个利器，打开VS Code左边的扩展栏 <font color="orange">⇧⌘X</font> ，然后输入chrome，选择并点安装 `Debugger for Chrome` 扩展，安装完后进入左边debug栏点击小齿轮 <font color="orange">F5</font> 在弹出的选择环境的下拉列表框中选择 chrome ，然后会自动打开Launch.json配置文件并有如下配置：

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "type": "chrome",
            "request": "launch",
            "name": "Launch Chrome against localhost",
            "url": "http://localhost:3000",
            "webRoot": "${workspaceFolder}"
        }
    ]
}
```
你必须指定一个 url 或者 file 配置项启动浏览器，以上配置的url就是指向我们最上面的debug-react项目的本地服务url。如果指定了url，就要设置webRoot字段，其表示一个绝对路径指示本地服务的文件来自哪里，比如以上的webRoot配置会解析`http://localhost:3000/index.js`成为类似` /Users/me/debug-react/app.js`所以确保它设置正确

在启动调试之前确保之前的本地服务是跑着的，然后按 <font color="orange">F5</font> 启动debug调试，可以看到VS Code启动了个新的浏览器窗口，如果程序并没有停在之前的断点处，可以在刷新一下调试 <font color="orange">⇧⌘F5</font> ，可以看到如下效果了：

 <div align=center><img width="70%" src="https://github.com/Cslove/Blog/raw/master/screenshots/debug-web.png"/></div>

 相应的props和相关变量都可以在debug栏清晰的看到，都不用去看浏览器的调试了。

 注意到上面的chrome调试配置的 request 类型是 launch，我们还可以尝试另一种调试方式，也就是`"request": "attach"`的调试方式，打开Launch.json配置文件，点击右下角的Add Cogfiguration选择`Chrome: attach`， VS Code自动生成了系列配置如下：

 ```json
{
    "type": "chrome",
    "request": "attach",
    "name": "Attach to Chrome",
    "port": 9222,
    "webRoot": "${workspaceFolder}"
}
 ```
要进行远程调试必须的打开chrome的远程调试功能，并设置以上默认配置的远程调试端口，如下：

- **Mac**：<br />在终端运行`/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome --remote-debugging-port=9222`， 会直接运行一个新的浏览器的窗口，然后将之前的debug-react项目重启一下`npm start`

- **Windows**：<br />右击桌面的chrome的快捷方式，点属性=>目标框的后面添加`--remote-debugging-port=9222`，或者在命令行下执行`<path to chrome>/chrome.exe --remote-debugging-port=9222`，重启一下debug-react项目

启动调试 <font color="orange">F5</font> ，可能会让你选一个本地服务，选择debug-react即可，可以看到已经开始了debug调试：

![image](https://github.com/Cslove/Blog/raw/master/screenshots/attach-debug.png)

点击以上绿色按钮重启一下，程序停在了断点的位置上了。注意attach并没有启动一个新的浏览器窗口，而是在你原有启动的窗口下开始了调试。将鼠标移到上图橙色正方图标上，显示的是断开连接，这也和之前我们所说的attach模式是把debugger连接到了已经启动的app上的说法一致，它没有像launch模式那样每次启动都会启一个新的浏览器窗口。

在调试react项目的时候会经常遇到render阶段数据还没出来，这跟react的生命周期有关，往往我们会多点几下单步调试或者单步跳过，其实打断点的时候支持多种断点，比如打条件断点，日志断点，函数断点等等...可以自行研究下～至此，调试web项目就到这了，下面我们来看一下怎样用此debugger科学的看那些库源码

## 源码调试

正如背景里面所提到的，在未接触VS Code的debugger之前，我看源码的过程真的很苦逼...而且时常搞不清源码中各个函数的调用栈顺序，效率着实慢的着急，由此往往没耐心往下看，也体会不到源码里的思想精髓...

废话不多说，这节我们以rollup源码为例。建议花半分钟可以去[官网](https://rollupjs.org/guide/en)看看这是个什么，不过看不看无所谓也不是这节的重点。建议把此项目clone下来跟着步骤一起走。

打开项目，发现整块源码都是用TypeScript写的（实践你会发现在VS Code中用debugger配合TypeScript看源码有多爽），VS Code可以调试任何可以编译成javascript的语言，更不用说亲儿子TS了。

![iamge](https://github.com/Cslove/Blog/raw/master/screenshots/rollup.png)

看到项目目录中是有自己的.vscode目录中，打开看下launch.json是有关于debug test的配置的，我们先点右下角添加一个Node.js: Launch配置，先放在这。然后看下package.json文件的scripts字段下的命令，在背景里面也说了在install完成后会执行build脚本，我们可以看看build中有一段`rollup -c`命令，这个命令就是典型的rollup打包命令，也就是说它以上一版本已经打包后的自已为依赖把自己的源码进行了打包，那我们就去看下rollup.config.js文件（-c 是指根据根目录下的rollup.config.js文件进行配置打包）。

打开rollup.config.js文件，主要是这三项配置：
```js
/* Rollup core node builds */
{
    input: 'src/node-entry.ts',
    ...,
    output: [
        { file: 'dist/rollup.js', format: 'cjs', sourcemap: true, banner },
        { file: 'dist/rollup.es.js', format: 'esm', banner }
    ]
},
/* Rollup CLI */
{
    input: 'bin/src/index.ts',
    ...,
    output: {
        file: 'bin/rollup',
        format: 'cjs',
        banner: '#!/usr/bin/env node',
        paths: {
            rollup: '../dist/rollup.js'
        }
    }
}
/* Rollup core browser builds */
{
    input: 'src/browser-entry.ts',
    ...,
    output: [
        { file: 'dist/rollup.browser.js', format: 'umd', name: 'rollup', banner },
        { file: 'dist/rollup.browser.es.js', format: 'esm', banner }
    ]
}
```

这是根据input文件打包到output的输出路径，可以看看package.json里面的files字段就对应与这里的output的输出路径，也就是最终发布到npm包里的所有代码。我们先从Rollup CLI打包过程看起，它的input输入文件是 bin/src/index.ts，这里的入口文件应该就是`rollup -c`对应的执行源文件了，我们先在`const command = minimist(...)`这行打个断点。

![image](https://github.com/Cslove/Blog/raw/master/screenshots/command.png)

既然bin/src/index.ts是最开始的`rollup -c`开始的执行源文件，那我们将此添加至刚刚在launch.json新添加的配置里

```js
{
    "type": "node",
    "request": "launch",
    "name": "Launch Program",
    "program": "${workspaceFolder}/bin/src/index.ts",  // 更换这里的路径
    "args": ["-c"]  // 传给program的参数
}
```

注意我们加➕了个args字段，这样就模拟了`rollup -c`命令，这时候若是按F5启动debug，会报个错
`无法启动程序.../src/index.ts，提示设置"outFiles"属性`。

就是说我们是以ts文件为入口的文件，VS Code需要编译后的js文件对源ts文件的映射，也就是需要sourcemap文件的路径，这样才能对应与源码的位置，可是我们看到只有dist目录下的rollup.js文件有对应的sourcemap文件，这从上面的三项配置就可以看到只有`Rollup core node builds`下的output下的file: 'dist/rollup.js'，设置了`sourcemap: true`，那我们给`Rollup CLI`也设置一下：

```js
/* Rollup CLI */
{
    input: 'bin/src/index.ts',
    ...,
    output: {
        file: 'bin/rollup',
        format: 'cjs',
        banner: '#!/usr/bin/env node',
        paths: {
            rollup: '../dist/rollup.js'
        },
        sourcemap: true    // 设置sourcemap
    }
}
```

这样给launch.json也要设置对应的 outFiles 属性`outFiles: ["${workspaceFolder}/bin/*"]`， 这样就能找到源文件的映射了。

我们改了rollup.config.js文件，那就意味着需要重新 run build 一下，在命令行执行`npm run build`，可以看到bin目录下有了新的rollup.map文件

然后F5启动debug，你会发现程序停在最开始打的command断点那个地方了，这样你就可以寻着debug的脚步 👣 探寻源码之旅了，大功告成！

## 彩蛋 🥚🥚
***
上面的源码调试虽然成功了，可是它调试的是原有的`npm run build`命令，也就是仓库自己的一些rollup配置，我想调试自己写的一些rollup配置该咋办勒。。。？

其实前面的背景已经提到了这个问题，在clone的rollup目录下执行`npm link`，然后你在你自己写的rollup配置目录里执行`npm link rollup`，这样你在node_modules（先安装好rollup依赖的其他依赖，可以先`npm install rollup`，然后删除node_modules里的rollup仓库，link你clone的rollup）里就可以看到整个rollup的链接仓库，在链接仓库里打断点，然后在按上面的步骤在你的例子项目里配置launch.json，这样启动你的debug，你会发现打的断点没用...😓，程序没有暂停过，这也是因为node_modules里的是symlinks，要使断点起作用的话，得给launch.json加上运行参数：

```js
{
    "runtimeArgs": [
        "--preserve-symlinks",
        // "--preserve-symlinks-main"
    ]
}
```

如果你的program指向的也是个链接文件就也要加上`"--preserve-symlinks-main"`参数，这样就能正常调试你自己写的配置了。（需要Node.js 10+）

***
在你调试的过程中进行单步调试的时候时常会进入一些node_modules里不想看的库，或者会进入到node的内置模块里，这些其实有时候没必要看，我们就可以跳过这些文件，例如：

```js
{
    "skipFiles": [
        "${workspaceFolder}/node_modules/**/*.js",
        "${workspaceFolder}/lib/**/*.js",
        "<node_internals>/**/*.js"   // node内部核心模块 设置的话在正常调试的时候可能有些问题。。。
    ]
}
```
***
在前面调试源码一节中我们修改了rollup源码的配置文件并重新手动 run build 了一下，其实我们可以直接在launch.json配置文件中配置一个`"preLaunchTask": "rollup"`，它表示在启动debug调试之前运行一个任务，可以在.vscode文件夹下新建一个task.json，针对源码调试那节我们可以写如下配置：

```js
{
    // See https://go.microsoft.com/fwlink/?LinkId=733558
    // for the documentation about the tasks.json format
    "version": "2.0.0",
    "tasks": [
        {
            "label": "rollup",
            "type": "shell",
            "command": "npm run build"
        }
    ]
}
```
然后在launch.json中配置`"preLaunchTask": "rollup"`，这样，我们就不用在debug之前手动去执行build，直接F5启动debug就行。相应的还有`postDebugTask`钩子在debug结束后执行一些任务，关于任务的具体设置信息可以看[这里](https://code.visualstudio.com/docs/editor/tasks)
***
#### 🎉🎉end🎉🎉

