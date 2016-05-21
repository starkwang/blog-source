title: 一些小技巧让JS代码更优雅
date: 2016-04-05 12:36:12
categories:
- 技术日记
------
今天翻了翻一年多前写的代码，感觉当年年轻的自己写下的代码真是图样啊（然而现在也没好到哪里去 /w\）。近期看了好多函数式编程以及设计模式的书和文章，于是想分享一些让JS代码更优雅的小技巧。

<!-- more -->
------
#一、善用函数式编程

假设我们有这样的需求，需要先把数组`foo`中的对象结构更改，然后从中挑选出一些符合条件的对象，并且把这些对象放进新数组`result`里。

```js
var foo = [{
	name: 'Stark',
	age: 21
},{
	name: 'Jarvis',
	age: 20
},{
	name: 'Pepper',
	age: 16
}]

//我们希望得到结构稍微不同，age大于16的对象：
var result = [{
	person: {
		name: 'Stark',
		age: 21
	},
	friends: []
},{
	person: {
		name: 'Jarvis',
		age: 20
	},
	friends: []
}]
```

从直觉上我们很容易写出这样的代码：

```js
var result = [];

//有时甚至是普通的for循环
foo.forEach(function(person){
	if(person.age > 16){
		var newItem = {
			person: person,
			friends: [];
		};
		result.push(newItem);
	}
})
```

然而用函数式的写法，代码可以优雅得多：

```js
var result = foo
	.filter(person => person.age > 16)
	.map(person => ({
		person: person,
		friends: []
	}))
```

还有比如在各种文章里说烂了的数组求和：

```js
var foo = [1, 2, 3, 4, 5];

//不优雅
function sum(arr){
	var x = 0;
	for(var i = 0; i < arr.length; i++){
		x += arr[i];
	}
	return x;
}
sum(foo) //15

//优雅
foo.reduce((a, b) => a + b) //15
```

这些只是很简单的例子，更多关于函数式编程的知识，可以参考这里：

[JS函数式编程指南 - GitBook](https://www.gitbook.com/book/llh911001/mostly-adequate-guide-chinese/details)


------
#二、lodash里一些很好用的东西
lodash是一个著名的JS工具库，里面存在众多函数式的方法和接口，在项目中引入可以简化很多冗余的逻辑。

[lodash中文文档](http://lodashjs.com/docs/)

###1、_.flow解决函数嵌套过深

```js
//很难看的嵌套
a(b(c(d(..args))));

//可以这样改善
_.flowRight(a,b,c,d)(..args)

//或者
_.flow(d,c,b,a)(..args)
```

###2、_.memoize加速数学计算

在写一些Canvas游戏或者其他WebGL应用的时候，经常有大量的数学运算，例如：

```js
Math.sin(1)
```
Math.sin()的性能比较差，如果我们对精度要求不是太高，我们可以使用`_.memoize`做一层缓存

```js
var Sin = _.memoize(function(x){
	return Math.sin(x);
})
Sin(1) //第一次使用速度比较慢
Sin(1) //第二次使用有了cache，速度极快
```

注意此处传入的`x`最好是整数或者较短的小数，否则memoize会极其占用内存。

**事实上，不仅是数学运算，任何函数式的方法都有可缓存性，这是函数式编程的一个明显的优点**

###3、_.flatten解构嵌套数组

```js
_.flatten([1, 2], [3, 4]); // => [1, 2, 3, 4]
```
这个方法和`Promise.all`结合十分有用处。

假设我们爬虫程序有个`getFansList`方法，它可以根据传入的值`x`，异步从粉丝列表中获取第 x*20 到 (x+1)\*20 个粉丝，现在我们希望获得前1000个粉丝：

```js
var works = [];
for (var i = 0; i < 50; i++) {
	works.push(getFansList(i))
}
Promise.all(works)
	.then(ArrayOfFansList=> _.flatten(ArrayOfFansList))
	.then(result => console.log(result))
```

前段时间写的[知乎关系网爬虫](https://github.com/starkwang/Zhihu-Spider/blob/master/src/fetchFollwerOrFollwee.js)中就能看到类似的写法

###4、_.once配合单例模式

有些函数会产生一个弹出框/遮罩层，或者负责app的初始化，因此这个函数是**执行且只执行一次**的。这就是所谓的单例模式，`_.once`大大简化了我们的工作量

```js
var initialize = _.once(createApplication);
initialize();
initialize();
// 这里实际上只执行了一次 initialize
// 不使用 once 的话需要自己手写一个闭包
```


#三、Generator + Promise改善异步流程

有时我们遇到这样的情况：

```js
getSomethingAsync()
	.then( a => method1(a) )
	.then( b => method2(b) )
	.then( c => method3(a,b,c) ) //a和b在这里是undefined！！！
```
只用 Promise 的话，解决方法只有把 a、b 一层层 return 下去，或者声明外部变量，把a、b放到 Promise 链的外部。但无论如何，代码都会变得很难看。

用 Generator 可以大大改善这种情况（这里使用了Generator的执行器co）：

```js
import co from 'co';

function* foo(){
	var a = yield getSomethingAsync();
	var b = yield method1(a);
	var c = yield method2(b);
	var result = yield method3(a,b,c);
	console.log(result);
}

co(foo());
```

当然，Generate 的用处远不止如此，在异步递归中它能发挥更大的用处。比如我们现在需要搜索一颗二叉树中value为100的节点，而这颗二叉树的取值方法是异步的（比如它在某个数据库中）

```js
import co from 'co';

function* searchBinaryTree(node, value){
	var nowValue = yield node.value();
	if(nowValue == value){
		return node;
	}else if(nowValue < value){
		var rightNode = yield node.getRightNode()
		return searchBinaryTree(rightNode, value);
	}else if(nowValue > value){
		var leftNode = yield node.getLeftNode()
		return searchBinaryTree(leftNode, value);
	}
}

co(searchBinaryTree(rootNode, 100))
```