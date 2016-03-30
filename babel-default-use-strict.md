title: babel6默认添加use strict引发的问题和解决方案
date: 2016-03-30
categories:
- ES2015
tags:
- ES2015 babel usestrict
---
*by 郭美青*

---
## 问题描述
我们有一个遗留系统，其中所有模块全部按照非strict模式，使用ES5语法书写。现在系统中新增一个全新的模块，我们希望能在这个新模块中引入ES2015语法来书写代码，因此引入了babel编译器来实现ES2015-> ES5的翻译，babel的版本是6.*。

在一次测试的时候我们发现Firefox的控制台报了这样一个错误：

	TypeError: access to strict mode caller function is censored

字面意思就是在[严格模式(strict mode)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Strict_mode)下使用了caller，这个不符合strict mode的要求，所以报错。

实际上，我们在书写ES2015模块时的确显式地在模块开始的第一行声明了:

	'use strict';

这些ES2015的模块里引用了一个遗留模块Ajax模块，这个模块里使用了caller这种写法,因此报错。

<!--more-->

## 问题定位
本以为是新增的ES2015模块声明的use strict的问题，直接删掉就可以解决问题，结果发现删掉后问题依旧，因此怀疑是否是Babel在转换时动了手脚。

因为项目中使用了Webpack来做构建，引入了babel-loader来做转换，所以通过看构建后的源码发现，虽然刚才手动删掉了自己写的'use strict',但是构建时又被自动的加上了。因此基本可以定位是babel搞的鬼。

google查了一下 关键字： babel use strict，找到这样一个页面：[How to remove global “use strict” added by babel](http://stackoverflow.com/questions/33821312/how-to-remove-global-use-strict-added-by-babel)

其中给出了问题和解决方案：
问题的原因是babel的es2015预设中默认使用了[babel-plugin-transform-strict-mode](https://babeljs.io/docs/plugins/transform-strict-mode/)，为所有模块自动加上**'use strict'**， 所以才有了上面的问题。

解决方案当然也呼之欲出：
对于Babel5有一种特殊的解决方案：

	//在.babelrc文件中加入这样一行配置：
	blacklist: ["useStrict"]


但是在Babel6中这个方案不生效，构建时会报错：

	Module build failed: ReferenceError: 
	[BABEL] E:\workspaces\branches\bundler\myapp\src\main\webapp\app.es6.js: Using removed Babel 5 option: 
	E:\workspaces\branches\bundler\myapp\src\main\webapp\.babelrc.blacklist - 
	Put the specific transforms you want in the `plugins` option

问题很明显，Babel6已经不支持blacklist这个option了，所以问题的答主给了一个babel6的解决方案：

因为babel给所有模块自动加'use strict'是通过[babel-plugin-transform-strict-mode](https://babeljs.io/docs/plugins/transform-strict-mode/)这个插件加的，因此在配置babel的时候不要使用es2015 的preset，自己手动指定plugins列表，把这个plugin排除在外不引进来就可以了。

查阅ES2015的preset发现压根没有这个plugins，在工程里的node_modules下搜索了一下，发现这个plugin是从[transform-es2015-modules-commonjs](http://babeljs.io/docs/plugins/transform-es2015-modules-commonjs/)这个插件里引入的。

至此终于定位到问题的根源了。

## 解决方案
由于无法限制ES2015模块依赖ES5的模块，所以理论上来说，只要是在ES2015模块里加了use strict，都会存在潜在的风险。所以最好的方式是去掉这个插件，或者再用一个插件移除掉所有的use strict，这样保证所有代码都在非严格模式下执行即可。

第一个方案显然是不靠谱了，除非不写import，否则必须要引入[transform-es2015-modules-commonjs](http://babeljs.io/docs/plugins/transform-es2015-modules-commonjs/)插件来做ES2015 -> commonJS Module 的转换，而引入了这个插件，就必须要引入[babel-plugin-transform-strict-mode](https://babeljs.io/docs/plugins/transform-strict-mode/)插件，这个很难在配置层面做到限制。

所以只能转向第二个方案：再写一个babel插件把所有的use strict干掉就可以了。习惯性地上github上搜索了一下，居然还真搜到了一个包：[babel-plugin-transform-remove-strict-mode](https://github.com/genify/babel-plugin-transform-remove-strict-mode)。果真是万能的Github！！！

粗略的看了下源码：

	exports["default"] = function () {
	    return {
	        visitor: {
	            Program: {
	                exit: function exit(path) {
	                    var list = path.node.directives;
	                    for(var i=list.length-1, it; i>=0 ; i--){
	                        it = list[i];
	                        if (it.value.value==='use strict'){
	                            list.splice(i,1);
	                        }
	                    }
	                }
	
	            }
	        }
	    };
	};

	module.exports = exports["default"];

再对比了一下[babel-plugin-transform-strict-mode](https://babeljs.io/docs/plugins/transform-strict-mode/)的源码：

	import * as t from "babel-types";

	export default function () {
	  	return {
		    visitor: {
		      Program(path, state) {
		        if (state.opts.strict === false) return;
		        
		        let { node } = path;
		
		        for (let directive of (node.directives: Array<Object>)) {
		          if (directive.value.value === "use strict") return;
		        }
		
		        path.unshiftContainer("directives", t.directive(t.directiveLiteral("use strict")));
		      }
		    }
	  	};
	}

源码我就不解读了，自己看吧。

确认无误后，在package.json文件中的devDependencies中加入

	 "babel-plugin-transform-remove-strict-mode":"0.0.2"

然后在.babelrc文件中新增配置：

	{ 
	  "presets": [
	    "es2015"
	  ] ,
	  "plugins":[
	    "transform-runtime",
		//下面这行
	    "transform-remove-strict-mode"
	  ]
	}

就把问题搞定了。

## 其他尚未确认的问题

- 这个问题仅出现在firefox中，chrome中并未出现，至于是何原因，仍未深究，有兴趣的同学可以继续深挖一下。
