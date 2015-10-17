title: 前端那些事儿（1） --- javascript模块化（上）
date: 2015-04-12 18:44:12
categories:
- javascript
------
最近在给小朋友做前端培训，发现大部分人对于框架、模块这些概念了解实在太少，就在这里写点东西给小朋友们科普一下。

所谓的模块，就是一段js代码，它可以实现页面里的各种功能，比如表单验证、进度条插件等等等等。下面我们不妨说说几种常见的写法：

<!-- more -->
----------


一、初阶写法
------

一个初阶的js脚本大概是这样子的：

```
function A(){
    //do something
}
function B(){
    //do something
}
A();
B();
```

这段脚本很简单，简单得不需要解释，声明了几个个函数，然后调用，这大概是很多初级的前端开发者的写法。

但这种写法有一个很严重的问题，那就是“全局变量污染”，在这个脚本里声明的函数都是全局函数，是可以在任何一处被访问甚至覆盖、重写。

这在小型的页面里当然不会出什么问题，但当会导致当js越来越多时，涉及到多人开发维护时，变量名就变得不可控，会出现太多的全局变量，这些全局变量都是地雷，随时会踩到。


----------


二、不那么初阶的写法（装进对象里）
-----------------

程序猿总是聪明的，既然要减少全局变量，那么我们把每个模块自己的函数放到各自的对象里不就行了么？
就像这样：

```
var module1 = {};
var module2 = {};

module1.A = function() {
	// body...
}
module1.B = function() {
	// body...
}

module2.A = function() {
	// body...
}
module2.B = function() {
	// body...
}
```

 显然我们声明了两个A函数和B函数，但因为他们是在各自的对象里面，所以不会冲突，嗯，每个人都很高兴。
 
但这样依然会有一个问题，那就是每个模块的变量全都是公开的，会暴露所有模块的成员，私有变量可能会被无意间修改，所以这是一种不那么安全的写法。

比如上面这个例子，我们随时可以重写module2.B，这可能会导致整个模块失效。

```
module2.B = 1;
```

----------


三、中阶写法（立即执行函数）
--------------

使用一个立即执行函数，可以完成不暴露私有变量的目的：

```
var module1 = (function() {
	var a = 12345;

	var f = function(){
		alert(a);
	}
	
	return {f : f};
})()
```

 不妨测试一下：

```
module1.f();   //12345
module1.a;   //undefined
```

当然还可以这样写：

```
(function() {
	//一些私有变量

	//下面就是局部的module1对象实现
	var module1 = {
		//somecode
	};

	//将局部module1提升到全局window下面
	window.module1 = module1;
	
})();
```

上面这两种就是最常见的js模块写法。

 例如我们经常使用的jQuery其实是这样实现的（来自jQuery2.0.3，剔除了无关代码）：

```
(function( window, undefined ) {
	//一些私有变量
	//。。。
	//。。。

	//下面就是局部的jQuery对象实现
	jQuery = function( selector, context ) {
		return new jQuery.fn.init( selector, context, rootjQuery );
	},

	//。。。
	//。。。
	//。。。

	//将局部的jQuery提升到全局window下面
	if ( typeof window === "object" && typeof window.document === "object" ) {
	window.jQuery = window.$ = jQuery;
}
})( window );
```

三、高阶写法（专业的模块工具）
---------------

既然上面这种写法如此完美，那为什么还需要像requireJS这种模块加载工具呢？

我们不妨想象一下，假设你的项目不使用模块加载工具，那么你的页面加载js的顺序应该是这样的：
1、jQuery、angularJS等一系列基础库
2、上面基础库的一系列插件
3、一些独立的小插件
4、你自己写的主要控制脚本

你的页面应该看起来像这样：
```
<script src="1.js"></script>
<script src="2.js"></script>
<script src="3.js"></script>
<script src="4.js"></script>
```

这有两个明显的缺点：

 1. 这个加载顺序非常严格，由于不同js模块之间的依赖，只要有一处写反，就会导致整体报错。当js文件变得越来越多的时候，这个依赖关系会变得非常复杂，不可维护。

 2. js的加载是阻塞式的，可能会导致白屏、卡顿等一系列问题。

所以我们需要一个好的加载工具来帮助我们实现js模块加载，于是就出现了requireJS这种模块加载工具。

你不需要纠结于脚本的加载顺序，只需要在模块前声明这个模块依赖了哪些模块，requireJS就会自动处理这些依赖关系，然后异步地把这些模块加载进来。

既然要声明并且自动加载，那就需要一个写模块的规范，但可惜的是，原生的JS并没有官方的模块规范，于是机智的程序猿就制定出了前端的JS模块规范，这就是我们说的AMD标准：
AMD：https://github.com/amdjs/amdjs-api/wiki/AMD

具体的requireJS用法会在下一篇文章中详细说明。









