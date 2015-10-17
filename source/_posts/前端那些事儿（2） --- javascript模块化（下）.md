title: 前端那些事儿（2） --- javascript模块化（下）
date: 2015-04-14 10:42:32
categories:
- javascript
------
上篇文章我们提到了原生方法加载JS模块时，因为存在阻塞、依赖关系复杂等等缺点，所以最好用加载工具来帮助我们完成加载，这就是requireJS、seaJS这样的加载工具，这篇文章暂时只关注于requireJS。

<!-- more -->
----------

引入requireJS
--

首先先去reqiureJS的官方网站下载：[http://requirejs.org/docs/download.html]

然后在HTML文档中引入：

```
<script data-main="js/main" src="js/require.js"></script>
```

为了防止阻塞，建议将script标签写到HTML文章的尾部，这样可以在DOM加载完之后再执行JS代码，避免白屏问题。

这里发现了一个奇怪的属性：data-main，是干什么用的？

这就是requireJS的入口，在加载完自身之后，requireJS会立即加载这个属性指向的文件，由于requireJS是默认加载JS文件的，所以可以在url里省略掉“.js”的后缀。


----------
## 主模块写法 ##

主模块长得大概像这个样子：

```
require.config({
    paths: {
        "module_1" : 'url',
        "module_2" : 'url',
        "module_3" : 'url'
    }
});
 
require(['module_1','module_2','module_3'], function(module_1,module_2,module_3) {
    //这里可以随意使用module下的功能
});
```

首先是config方法，这个方法定义了所有模块的名字、url，以便加载时使用。

然后就是喜闻乐见的require方法了，这个方法有两个参数：

 - 第一个参数是依赖的模块名称，注意是string类型的，会有很多人忘记这一点。
 - 第二个参数是对应执行的函数。

require方法会自动解析模块的依赖关系，比如3依赖2，2依赖1，那么在加载3模块之前，requireJS会提前加载并且运行1模块和2模块，非常方便。


----------
## 如何写模块 ##

比如我们定义了一个foo模块，它有一个方法a，并且他是依赖于jQuery的，：

```
define(["jQuery"], function() {
  return {
    a: function() {
	  //可以在这里使用jQuery
      $("div").css("display","none");
    }
  }
});
```

这就是基础的模块写法。


----------

其他参考资料
------
requireJS在网上的资源很多，所以这篇文章并不长，因为我觉得我的文笔水平不如其他博客大大。

这里给出了一些可以参考的资料：
http://www.tuicool.com/articles/jam2Anv
http://www.ruanyifeng.com/blog/2012/11/require_js.html
http://www.cnblogs.com/snandy/archive/2012/05/22/2513652.html

