## 语法规范
- 对ES6 Module的原生支持。

    不再需要babel的转换了，但这并不意味着模块的其他代码使用es6语法编写不需要babel的编译。webpack2仅仅支持module的转换


- 支持ES6语法的代码拆分方式。
    
    ES6 Loader规范定义了`System.import` 方法用于运行时动态加载ES6模块，Webpack2把`System.import`定义为类似require.ensure的拆分点，生成独立的chunk。另外，chunk加载失败产生的错误可以被捕获了。

    - `System.import` 由于不符合新提出的标准，webpack在v2.1.0-beta28版本中被废弃，转向支持`import`。
    
    ```js
    //好处是向标准看齐，果断把require.ensure 改成import吧
    import("./module").then(module => {
      return module.default;
    }).catch(err => {
      console.log("Chunk loading failed");
    });
    ```
    - 由于这个建议还在 Stage 3，如果你想要同时使用 import 和 Babel，你需要安装/添加 dynamic-import 语法插件来绕过解析错误。当建议被添加到规范之后，就不再需要这个语法插件了。
    - require.ensure支持手动设置第三个参数来对chunk进行自定义命名，目前import尚不支持。如果要使用该功能，请继续使用`require.ensure`。
    
    
    
- 动态表达式

    import支持动态表达式参数，处理的方式是会把所有的可能性都生成独立的chunk。
    这个貌似在webpack1.x的时候已经实现了require的动态表达式功能，不知道为何又提起？
    
- ES6、AMD、CommonJS混用
    
    工程里，不同的模块内可以使用不同的模块化规范，甚至同一模块内也可以同时支持多种模块化规范的混用

- Babel和Webpack

    默认情况下，Babel的es2015 preset会把ES6 module语法转换成commonJS格式，如果希望Webpack直接处理ES6 Module 语法，babel预设应该换用es2015-webpack。
    
    ```js
    // es2015-webpack预设没什么用啊，预设配置成
    {
      "presets": [
        ["es2015", { "modules": false }]
      ]
    }
    // 才有效果。
    ```
- 针对ES6规范的优化

    ES6规范有与生俱来的静态分析性能优势，webpack2可以根据集中规范的import写法来分析出导入的这个模块是否被使用，如果没有被使用，就压缩或者删除掉。
    
- ES6的pollyfill

    1. Promise-pollyfill
    2. Object.defineProperty
    3. Function.prototype.bind
    4. Object.keys
    
- [resolve 选项的更新](https://github.com/webpack/enhanced-resolve)

    基本上是配置的简化，变得更加不容易写错。
    
## loader配置
- loaders已经改为rules。
- loader不能再省略-loader后缀了，必须手动添加。当然也可以通过下面配置来向前兼容，但是不被推荐。
    ```js
    resolveLoader:{moduleExtensions: ["-loader"]}
    ```
- 取消了module.preLoaders以及module.postLoaders配置
- json-loader 不再需要手动添加，webpack会自动尝试使用该loader来加载json
- 匹配文件的逻辑变成了只匹配资源路径，不再匹配资源。意味着query string不再参与匹配了。

```
# 如果要匹配下面这个url
background:url('./asset/img/a.png?v=1')

# loader的test要写成 /\.png($|\?)/
# 现在就直接写成/\.png$/就可以了

```
- 配置文件所指定的各个loader现在是相对于配置文件进行解析的（但如果配置文件指定了 context选项则以它为准）。在以前，如果某些外部模块是通过 npm link链接到当前包的，则会产生问题，现在应该都可以解决了。

    ```js
     module: {
        rules: [
          {
            // ...
    -       loader: require.resolve("my-loader")
    +       loader: "my-loader"
          }
        ]
      },
      resolveLoader: {
    -   root: path.resolve(__dirname, "node_modules")
      }
    ```

- Loader配置中经常会有一些loader需要读取一些特定的参数或者配置，比如context、debug等。webpack1都是通过在每个loader自己的query或者options里，或者在loader的配置中设置。现在统一改成通过LoaderOptionsPlugin插件来注入了。这么设计的目的是出于“关注点分离”的考虑。webpack的根配置中应该禁止添加自定义的字段名，以便让校验配置成为可能。

    以前：

```js
- context: __dirname,
- debug: true,
plugins:[
+  new webpack.LoaderOptionPlugin({
+      options: {
+          context: __dirname
+      }
+  })
]
```
    
## 压缩
- UglifyJsPlugin 将不再把所有 loader 都切到代码压缩模式。debug选项已经被移除。Uglify的sourceMap默认设置为false，需要手动开启。
    ```js
    devtool: "source-map",
      plugins: [
        new UglifyJsPlugin({
    +     sourceMap: true,
    //warning默认关闭，需要手动开启
    +     compress:{
    +       warnings: true
    +     }
        })
      ]
    ```
- UglifyJsPlugin不再压缩loaders。计划在webpack3或者更高的版本中取消。当前如果要用可以参照下面配置：

```js
 plugins: [
+   new webpack.LoaderOptionsPlugin({
+     minimize: true
+   })
  ]
```
    
**插件**
- 插件新增了一个options对象，不再建议把配置写成参数形式

    ```js
    new ExtractTextPlugin(
      '[name].css', 
    - {
	-   allChunks: false
	- }
	+ options: {
	+   allChunks: false
	+ }
	)
	```
- OccurrenceOrderPlugin这个插件已经不再需要，默认开启。这个插件的作用是对模块的id命名按照出现的顺序来命名
- ExtractTextWebpackPlugin1.0不能再webpack v2下工作，必须要明确地升级到v2
```sh
npm install --save-dev extract-text-webpack-plugin@beta
```

## 其他
- require.ensure 和AMD的require的加载方式现在都是异步了，即便chunk已经加载。
- 热更新的通信机制由Web Messaging API（postMessage）更换为webSocket。

    因此webSocket必须内联到打包文件中。webpack-dev-server默认出于内联模式。这应该使得我们可以用webpack-dev-server来更新WebWorker中的代码。
    ```
    // 解释webworker这一句，todo
    ```

- 在[CLI](https://webpack.js.org/api/cli/)中传自定义参数策略有变化。
    
    `webpack --custom-flag` 这种做法在webpackV2中不被允许了。替代方案是：

    `webpack --env.customFlag`
    ```js
    module.exports = function(env){
        var customFlag = env.customFlag;
        //...
        
    }
    ```
