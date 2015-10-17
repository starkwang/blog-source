title: JSONP小结
date: 2015-06-21 20:55
categories:
- 技术日记
------
## JSONP是什么 ##

JSONP(JSON with Padding)是JSON的一种“使用模式”，可用于解决主流浏览器的跨域数据访问的问题。

由于同源策略，一般来说位于 a.com 的网页无法与不是a.com的服务器沟通，而 HTML 的`<script>` 元素是一个例外。

利用 `<script>` 元素的这个开放策略，网页可以得到从其他来源动态产生的 JSON 资料，而这种使用模式就是所谓的 JSONP。用 JSONP 抓到的资料并不是 JSON，而是任意的JavaScript，用 JavaScript 直译器执行而不是用 JSON 解析器解析。
<!-- more -->

----------

## 为什么需要JSONP ##
我们在平时的开发过程中可能遇到这种问题，某张网页的域名位于a.com，但我们需要从b.com这个url那里拉取数据。

这个时候使用Ajax是无法获取数据的，因为Ajax为了网页的安全性使用了“同源策略”。域名不同的url之间是无法使用ajax直接拉取数据的。

那么怎么办呢？


----------
## JSONP的解决方法 ##

**一、简单例子**

首先我们假设服务器端有一个这样的data.js文件：

```
//data.js

var person = {
	name : "stark" , 
	sex : "male"
}
```
然后我们在html中引用它，同时我们再写一个handler函数：

```
<script src="data.js"></script>

<script>
	function handler(data){
		alert(data.name);
	}

	handler(person);//此处可以调用data.js里面的数据
</script>
```
注意到这里的data.js实际上可以在任意的域名下，这就意味着我们可以从任意的域名拉取data.js，实现跨域传递数据的效果。


**二、改进方法**

上面的方法存在局限性。

首先，data的引用是写死在页面中的。这意味着我们无法“动态”地获取数据。

其次，handler的调用也是直接写在js里面，不仅需要一个全局变量person，还需要严格地保证函数调用时数据已经data.js进来，否则会报错。

所以我们不妨这样改进一下：

首先写一个handler函数：
```
function handler(data){
	alert(data.name);
}
```
然后拉取数据的步骤如下：

```
var script = document.createElement('script');

script.setAttribute('src',
'http://b.com/somePage?callback=handler&data=person');

document.getElementsByTagName('head')[0].appendChild(script);
```
我们在这里添加了一个`<script>`节点，它的url是`http://b.com/somePage?callback=handler&data=person`，这显然是一个get请求的url。

我们告诉服务器，回调函数为handler，我们需要的是person的数据，然后服务器返回类似这样的字符串：

```
handler({
	name : "person" , 
	sex : "male"
});
```

等浏览器获取到这堆字符串，便会把它当做js脚本立即执行，于是跨域获取数据就达成了。

**三、jQuery的方法**

上面的方法是自己从零开始写的，封装性和复用性很差，幸运的是jQuery已经为我们做好了这些东西。

```
$.ajax({
             type: "get",
             
             async: false,
             
             url: "http://b.com/somePage?data=person",
             
             dataType: "jsonp",
             
             jsonp: "callback",//传递给请求处理程序或页面的，用以获得jsonp回调函数名的参数名(一般默认为:callback)
             
             success: function(json){
                 alert(json.name);
             },
             
             error: function(){
                 alert('fail!');
             }
         });
```
这里没有写handler这个函数，而是把回调写在了success里，但也能运行成功。

这就是jQuery的功劳了，jquery在处理jsonp类型的ajax时（但其实它们真的不是一回事儿），自动生成了一个回调函数并把数据取出来供success属性方法来调用。





