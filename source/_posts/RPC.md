title: 一个浏览器和NodeJS通用的RPC框架
date: 2016-05-11 10:02:03
tags:
- 技术日记
---

[starkwang/Maus: A Simple JSON-RPC Framework running in NodeJS or Browser, based on websocket.](https://github.com/starkwang/Maus)

这几天写了个小型的RPC框架，最初只是想用 TCP-JSON 写个纯 NodeJS 平台的东西，后来无意中开了个脑洞，如果基于 Websocket 把浏览器当做 RPC Server ，那岂不是只要是能运行浏览器（或者nodejs）的设备，都可以作为分布式计算中的一个 Worker 了吗？

打开一张网页，就能成为分布式计算的一个节点，看起来还是挺酷炫的。

<!-- more -->

#一、什么是RPC
可以参考：
[谁能用通俗的语言解释一下什么是RPC框架？ - 知乎](https://www.zhihu.com/question/25536695)

简单地说就是你可以这样注册一个任意数量的`worker`（姑且叫这个名字好了），它里面声明了具体的方法实现：

```js
var rpcWorker = require('maus').worker;
rpcWorker.create({
    add: (x, y) => x + y
}, 'http://192.168.1.100:8124');

```
然后你可以在另一个node进程里这样调用：

```js
var rpcManager = require('maus').manager;
rpcManager.create(workers => {
	workers.add(1, 2, result => console.log(result));
}, 8124)
```
这里我们封装了底层的通信细节（可以是tcp、http、websocket等等）和任务分配，只需要用异步的方式去调用`worker`提供的方法即可，通过这个我们可以轻而易举地做到分布式计算的`map`和`reduce`：

```js
rpcManager.create(workers => {
	//首先定义一个promise化的add
	var add = function(x, y){
		return new Promise((resolve, reject)=>{
			workers.add(x, y, result => resolve(result));
		})
	}
	//map&reduce
	Promise.all([add(1,2), add(3,4), add(4,5)])
		.then(result => result.reduce((x, y) => x + y))
		.then(sum => console.log(sum)) //19
}, 8124)
```

如果我们有三个已经注册的`Worker`（可能是本地的另一个nodejs进程、某个设备上的浏览器、另一个机器上的nodejs），那么我们这里会分别在这三个机器上分别计算三个`add`，并且将三个结果在本地相加，得到最后的值，这就是分布式计算的基础。

------
#二、Manager的实现

###0、通信标准
要实现双向的通信，我们首先要定义这样一个“远程调用”的通信标准，在我的实现中比较简单：

```js
{
	[id]: uuid          //在某些通信中需要唯一标识码
	message: '......'   //消息类别
	body: ......        //携带的数据
}
```


###1、初始化
首先我们要解决的问题是，如何让`Manager`知道`Worker`提供了哪些方法可供调用？

这个问题其实很简单，只要在 websocket 建立的时刻发送一个`init`消息就可以了，`init`消息大概长这样：

```js
{
	message: 'init',
	body: ['add', 'multiply'] //body是方法名组成的数组
}
```

同时，我们要将`Manager`传入的回调函数，记录到`Manager.__workersStaticCallback`中，以便延迟调用：

```js
manager.create(callback, port) //记录下这个callback

//一段时间后。。。。。。

manager.start() //任务开始
```

###2、生成workers实例
现在我们的`Manager`收到了一个远程可调用的方法名组成的数组，我们接下来需要在`Manager`中生成一个`workers`实例，它应该包含所有这些方法名，但底层依然是调用一个webpack通信。这里我们可以用类似元编程的奇技淫巧，下面的是部分代码：

```js
//收到worker发来的init消息之后
var workers = {
    __send: this.__send.bind(this), //这个this指向Manager，而不是自己
    __functionCall: this.__functionCall.bind(this) //同上
};
var funcNames = data.body; //比如['add', 'multiply']
funcNames.forEach(funcName => {
	//使用new Function的奇技淫巧
	rpc[funcName] = new Function(`
        //截取参数
        var params = Array.prototype.slice.call(arguments,0,arguments.length-1);
        var callback = arguments[arguments.length-1];
        
        //这个__functionCall调用了Manager底层的通信，具体在后面解释
        this.__functionCall('${funcName}',params,callback);
    `)
})
//将workers注册到Manager内部
this.__workers = workers;
//如果此时Manager已经在等待开始了，那么开始任务
if (this.__waitingForInit) {
    this.start();
}
```

还记得上面我们有个`start`方法么？它是这样写的：

```js
start: function() {
    if (this.__workers != undefined) {
        //如果初始化完毕，workers实例存在
        this.__workersStaticCallback(this.__workers);
        this.__waitingForInit = false;
    } else {
        //否则将等待初始化完毕
        this.__waitingForInit = true;
    }
},
```

###3、序列化
如果只是单个`Worker`和单个`Manager`，并且远程方法都是同步而非异步的，那么我们显然不需要考虑返回值顺序的问题：

比如我们的`Manager`调用了下面一堆方法：

```js
workers.add(1, 1, callback);
workers.add(2, 2, callback);
workers.add(3, 3, callback);
```

由于`Worker`中`add`的是同步的方法，那么显然我们收到返回值的顺序是：

```
2
4
6
```

但如果`Worker`中存在一个异步调用，那么这个顺序就会被打乱：

```
workers.readFile('xxx', callback);
workers.add(1, 1, callback);
workers.add(2, 2, callback);
```

显然我们收到的返回值顺序是：

```
2
4
content of xxx
```
所以这里就需要对发出的函数调用做一个序列化，具体的方法就是对于每一个调用都给一个uuid（唯一标识码）。

比如我们调用了：

```js
workers.add(1, 1, stupid_callback);
```
那么首先`Manager`会对这个调用生成一个 uuid ：

```
9557881b-25d7-4c94-84c8-2463c53b67f4
```

然后在`__callbackStore`中将这个 uuid 和`stupid_callback `绑定，然后向选中的某个`Worker`发送函数调用信息（具体怎么选`Worker`我们后面再说）：

```js
{
	id: '9557881b-25d7-4c94-84c8-2463c53b67f4',
	message: 'function call',
	body: { 
		funcName: 'add', 
		params: [1, 1] 
	}
}
```
`Worker`执行这个函数之后，发送回来一个函数返回值的信息体，大概是这样：

```js
{
	id: '9557881b-25d7-4c94-84c8-2463c53b67f4',
	message: 'function call',
	body: { 
		result: 2 
	}
}
```

然后我们就可以在`__callbackStore`中找到这个 uuid 对应的 callback ，并且执行它：

```js
this.__callbackStore[id](result);
```

这就是`workers.add(1, 1, stupid_callback)`这行代码背后的原理。
###4、任务分配
如果存在多个`Worker`，显然我们不能把所有的调用都傻傻地发送到第一个`Worker`身上，所以这里就需要有一个任务分配机制，我的机制比较简单，大概说就是在一张表里对每个`Worker`记录下它是否繁忙的状态，每次当有调用需求的时候，先遍历这张表，

1. 如果找到有空闲的`Worker`，那么就将对它发送调用；
2. 如果所有`Worker`都繁忙，那么先把这个调用暂存在一个队列之中；
3. 当收到某个`Worker`的返回值后，会检查队列中是否有任务，有的话，那么就对这个`Worker`发送最前的函数调用，若没有，就把这个`Worker`设为空闲状态。

具体任务分配的代码比较冗余，分散在各个方法内，所以只介绍方法，就不贴上来了/w\

全部的Manager代码在这里（抱歉还没时间补注释）：

[Maus/manager.js at master · starkwang/Maus](https://github.com/starkwang/Maus/blob/master/src/manager.js)

#三、Worker的实现

这里要再说一遍，我们的RPC框架是基于websocket的，所以`Worker`可以是一个PC浏览器！！！可以是一个手机浏览器！！！可以是一个平板浏览器！！！


`Worker`的实现远比`Manager`简单，因为它只需要对唯一一个`Manager`通信，它的逻辑只有：

1. 接收`Manager`发来的数据；
2. 根据数据做出相应的反应（函数调用、初始化等等）；
3. 发送返回值

所以我们也不放代码了，有兴趣的可以看这里：

[Maus/worker.js at master · starkwang/Maus](https://github.com/starkwang/Maus/blob/master/src/worker.js)



#四、写一个分布式算法
假设我们的加法是通过这个框架异步调用的，那么我们该怎么写算法呢？

在单机情况下，写个斐波拉契数列简直跟喝水一样简单（事实上这种暴力递归的写法非常非常傻逼且性能低下，只是作为范例演示用）：

```js
var fib = x => x>1 ? fib(x-1)+fib(x-2) : x
```

但是在分布式环境下，我们要将`workers.add`方法封装成一个Promise化的`add`：

```js
//这里的x, y可能是数字，也可能是个Promise，所以要先调用Promise.all
var add = function(x, y){
	return Promise.all([x, y])
		.then(arr => new Promise((resolve, reject) => {
			workers.add(arr[0], arr[1], result => resolve(result));
		}))
}
```

然后我们就可以用类似同步的递归方法这样写一个分布式的`fib`算法：

```js
var fib = x => x>1 ? add(fib(x-1), fib(x-2)) : x;
```
然后你可以尝试用你的电脑里、树莓派里、服务器里的nodejs、手机平板上的浏览器作为一个`Worker`，总之集合所有的计算能力，一起来计算这个傻傻的算法（事实上相比于单机算法会慢很多很多，因为通信上的延迟远大于单机的加法计算，但只是为了演示啦）：

```js
//分布式计算fib(40)
fib(40).then(result => console.log(result));
```

