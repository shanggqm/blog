# webpack学习笔记
----
## [写一个webpack插件](https://webpack.github.io/docs/how-to-write-a-plugin.html)
写webpack插件，最重要的是要理解两个概念：`compiler`和`compilation`

[compiler](https://github.com/webpack/webpack/blob/master/lib/Compiler.js)是webpack运行时全局的环境变量，启动webpack时就会根据用户的所有配置进行创建，并在使用插件时把compiler的引用传递给插件，以便插件可以访问webpack全局的运行时配置和状态。

[compilation](https://github.com/webpack/webpack/blob/master/lib/Compilation.js)对象和每一次build对应，代表一组有最新版本号的asset集合。每当运行webpack 或者每次监听到文件改变都会创建一个对应的compilation。每一个compilation都包含了对应build所包含的模块信息、文件变化信息、以及依赖信息和编译后的assets信息。

