#SPA架构下，hash传递UGC类参数时由于编码引发的问题

##问题描述
在SPA单页面架构中，通过hash传递name, desc这种UGC字段时，因为无法控制特殊字符导致在解析hash后面的参数时出错。比如:http://hao.a.com/init.action#main/module/create/~id=12323&name=alsdf@@#$%)^)&@$%
 
因为目前hash结构是#/page/entity/crud/~param，实际开发中会有通过hash获取参数的需求，尤其在页面直接通过URL定位、刷新或者链接跳转时，因为没有上下文的JS参数传递，如果不借助接口传参的话，则只能通过url去拿参数。
 
以上说的参数传递问题，理论上是可以通过良好的接口设计来避免，但实际开发中，由于缺乏统一的规范，需要和后端协调商定参数等问题，很难成功推行。因此还需要前端方案自己来保障及时有类似的需求，也可以正确、稳定、安全的传递参数。

##方案设计
系统中的参数传递，一般会涉及到两种：`id类`，`UGC类`
- id类是比较常见的参数，一般由数据库生成，后端返回给前端，而且一般是纯数字，结构简单，且不存在安全问题；
- UGC类是由用户在系统中输入并存储到DB中的字段，通常会存在xss风险，特殊字符问题等等，属于不安全内容，必须要对该类字段进行有效的处理；
 
UGC类内容的使用场景一般分为两种：
- 从后端取回，显示在界面中
- 被当成参数，通过url进行传递

第一种场景，要注意xss注入的风险，需要对尖括号，或者不安全字符进行escape；或者通过模板库渲染，杜绝手动设置innerHTML，或者jquery.html这种使用方法；

第二种场景的情况比较多：
* 可以通过模板直接生成在a标签上;

```html
<a href="#main/module/crud/~id={{id}}&name={{name}}">新建</a>
```
* 可以通过js动态赋值到href上

```javascript
var id = 12;
var name = '帆布鞋';

$('#button').attr('href', '#main/module/crud/~id=' + id + '&name=' + name);
```

* 可以直接拼凑hash url，直接navigate
```javascript
var id = 12;
var name = '帆布鞋';

var url = 'main/module/crud/~id=' + id + '&name='+ name;
router.navigate(url, config);
```
这些情况处理逻辑各不相同，有些可以很方便的对参数做encodeURIComponent，有些情况却不容易做，比如第一种通过蒙版生成a标签的方式因此统一出口逻辑很重要，比如在全局router下定义一个静态function nav(){} 用来执行所有通过hash路由导航的功能：

```javascript
function nav(url, params, navConfig){
	var hash = url + '~',
	    paramsArray = [];
	for(var e in params){
		if(params.hasOwnProperty(e)){
			paramsArray.push(e + '=' + encodeURIComponent(params[e]));
		}
	}
	hash = hash + paramsArray.join('&');
	this.navigate(hash, navConfig || this.navConfig);	
}
```

上述方法对所有的参数统一采用url encode，再放到url的hash里，理论上讲只要通过这种方式来路由，可以避免上文提到的参数解析问题，实际的URL可能是这样：
http://hao.a.com/init.action#main/page/crud/~id=12&name=%25E6%25B5%258B%25E8%25AF%2595%25E6%258E%25A8%25E5%25B9%25BF%25E8%25AE%25A1%25E5%2588%2592!%2540%2523%2524%2523%2523%2524%255E%2525%2526%2524%2525%2526%2523%2525%255E%2526%2523%2525%255E%2523%2526*%2525

通过解析/~后面的参数字符串，先分割，再decode，理论上可以正确拿到各个参数值，比如下面的parseParams方法

```javascript
function parseParam(param){
  var p = {};
  if(!param){
    return p;
  }
  param = param.split('&');
  for(var i = 0; i< param.length; i++){
    var kv = param[i].split('=');
    p[kv[0]] = decodeURIComponent(kv[1]);
  }
  return p;
}
```
问题看似已经完美的解决了，实际环境中一跑，发现仍然无法正常解析。什么原因呢？通过debug发现，backbone在navigate函数里，对hash做了一些处理，如下图的_extractParameters方法

![_extractParameters](https://github.com/shanggqm/blog/blob/master/asset/image/20150827_paper1_1.png "_extractParameters")

这个方法对自定义hash结构中的参数做了decodeURIComponent的解码操作，如下图所示：

![backbone](https://github.com/shanggqm/blog/blob/master/asset/image/20150827_paper1_2.png "backbone")

所以实际上，在Backbone Router里设置的自定义路由参数：

```javascript
routers:{
  'main/:module/(:crud/)(~:param)': 'main'
}
```

中的param解析后就是已经解码过的参数：
http://hao.a.com/init.action#main/module/crud/~id=131201&name=特殊字符@#$##$^%&$%&#%^&#%^#&*%
此时再对/~后面的param做parseParam分割就会出错，因为name中包含了'&'字符。

问题找到了，解决的方法也很简单，在navigate的时候，对参数做两次encodeURIComponent即可。

```javascript
function nav(url, params, navConfig){
	var hash = url + '~',
	    paramsArray = [];
	for(var e in params){
		if(params.hasOwnProperty(e)){
			paramsArray.push(e + '=' + encodeURIComponent(encodeURIComponent(params[e])));
		}
	}
	hash = hash + paramsArray.join('&');
	this.navigate(hash, navConfig || this.navConfig);	
}
```

##经验总结
- 实际开发的时候，为了杜绝上述问题的再次发生，最好从规范层面来解决，不允许直接通过模板的方式给a标签设置hash url。统一采用js捕获点击事件来执行nav操作。
- 对于这种和底层框架相关、具备通用性的问题，最好在底层解决，不要放到业务代码层面去自己encode。
