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