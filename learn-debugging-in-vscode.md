# Debugging in VSCode
 <!-- ![image](https://github.com/Cslove/Blog/raw/master/screenshots/pic.jpg) -->
 ## 背景

之前在看一些github源码的时候看调试数据是非常非常吃力，不知道多少时间消耗在了console，rebuild，switch tab的这几个频率最高的步骤中，着实烦躁。首先回顾一下我自己之前在看一些库源码的过程，以rollup这个打包库为例，首先`git clone`下来，`npm install`后发现它也自动执行了打包过程，看了下`package.json`文件中的scripts字段，其中有prepare脚本命令，npm在执行`npm install`后就会自动执行这个script，这个命令又指向了`npm run build`，进而进行了自动打包过程。通常我都会在这个project下执行`npm link`然后重新重建个文件夹写自己的一些例子项目link到前面的project，这样看project下的源码或者console之后重新build，就能在例子项目实时能够调试到，其中执行`npm run watch`可以不用重新build。

这样真的很累！...需要开两个项目频繁切换，消耗时间，还一点都不灵活！下面就研究了下VSCode中强大的Debug功能，让我犹如坐上了火箭🚀，并在此做个总结。

## 初识Debug Panel
下面是vscode中的debug panel

 ![image](https://code.visualstudio.com/assets/docs/editor/debugging/debugging_hero.png)

 最左边一列的第四个tab，之前的我看到过无数遍这个小蜘蛛，却从来没有打开过...，相信我，它会让你打开新世界的大门。