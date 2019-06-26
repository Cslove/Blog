# Debugging in VSCode
 <!-- ![image](https://github.com/Cslove/Blog/raw/master/screenshots/pic.jpg) -->
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

  ![image](https://code.visualstudio.com/assets/docs/editor/debugging/launch-configuration.png)

> *如果你的项目根目录下有.vscode目录且下面有launch.json配置文件，则点击上面的小齿轮按钮会直接打开此launch.json配置文件*

这时会出现如下，让你选择调试环境，VSCode内置了Node.js环境，可以在插件tab安装其他语言的扩展，VSCode支持各种语言的调试，eg：PHP，Python，go，C/C++...我们直接选择Node.js环境

  ![image](https://code.visualstudio.com/assets/docs/editor/debugging/debug-environments.png)

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

![image](https://github.com/Cslove/Blog/raw/master/screenshots/launch.png)

它会自动帮我们生成该配置的常用项，我们将其中的`"program": "${file}"`改为`"program": "${workspaceRoot}/app.js"`，在这个配置有挺多的内置变量可以直接使用，`${file}`就是当前所活跃的文件，`${workspaceRoot}`表示当前项目的工作根目录，完整的替换变量的列表可参考[这里](https://code.visualstudio.com/docs/editor/variables-reference)

> *在上面的配置字段中有个`request`字段，有两个值可以选择：`"launch"`和 `"attach"`，它表示VS Code中核心的两种调试模式。当时搞清这两种模式区别的时候也是有点晕，这也是取决与你项目中的工作流是怎样的，如果你调试的是web项目，通常你会在浏览器中已经直接打开本地项目了，这个时候我们就要使用attach模式，正如字面的意思，将debugger附加到你已经运行到的web项目，而launch就像字面意思是直接由编辑器启动这个程序，比如启动一个服务端项目或者上面我们的小例子，这个启动模式编辑器会自动把debugger附加到这个程序中，下面我们在源码调试和web项目调试两节会有这两种模式的直观印象*