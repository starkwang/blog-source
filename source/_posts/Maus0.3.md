title: Maus0.3 发布
date: 2016-05-20 19:01:56
categories:
- 技术日记
---

本来应该早几天就写完的，结果拖延症到现在。

[GitHub - starkwang/Maus: A Light RPC Framework for NodeJS or Browser.](https://github.com/starkwang/Maus)


Maus 0.3新特性如下：

1. 当任意Worker连接失去时，Manager会把已发出给这个Worker但未完成的函数调用重新发送到其它可用的Worker上。简单地说即使，现在的框架允许任意Worker失联（当然起码要有一个Worker啦），大大提高了容错性。
2. 加入了Parkserver（职介所）的新特性，下面详细说说Parkserver是啥。


<!-- more -->
------
#Parkserver
一个最简单的分布式框架大概就是一个Manager和若干个Worker，Manager对Worker发出函数调用，Worker执行完成后把结果返回至Manager进行下一步处理。

![](http://img.blog.csdn.net/20160521125613971)

这种模式的问题在于Manager和Worker是强耦合的，尤其是Manager，一旦它停止工作，那么所有的Worker只会傻傻地等待Manager回归网络。如果我们存在多个Manager，那么只能很不智能地手工分配Worker（因为连接Manager需要在代码中写入Manager的地址）。

所以我们需要一个类似『职介所』的东西，所有的Worker不再直接连接Manager，而是到一个『职介所』注册自己的信息。职介所负责调度Worker，一旦收到Manager请求Worker的信息，那么它会搜寻合适的Worker，并且让这些Worker连接到Manager。

1. Worker向职介所注册自己的信息；
2. Manager向职介所请求Worker（可以指定数量、类型）；
3. 职介所根据已经注册的信息，挑选合适的Worker，向这些Worker发送Manager的地址，把这些Worker的状态设为unavaliable；
4. 被选中的Worker连接到Manager；
5. Manager如果停止工作，那么被选中的Worker会通知职介所把自己的状态重新更新为avaliable。

![](http://img.blog.csdn.net/20160521125729644)

这样有什么好处呢？

第一，Worker在这种模式下更像是一种『计算资源』，而不是『节点』，它可以被Parkserver分配到任何Manager上。

第二，新的Manager可以随时加入计算网络，像Parkserver请求计算资源后，执行自己的任务。执行完任务后，可以释放自己的Worker。

第三，新的Worker也可以随时加入计算网络，与以往不同的是，加入计算网络时不再需要预先设置好自己的Manager，而只需要向Parkserver注册即可。

------
#API
说了这么多，下面直接看新的接口吧：

现在你可以这样开启一个Parkserver：

```js
var parkserver = require('maus').parkserver;
var myParkserver = new parkserver(8500);
```

然后Worker中这样连接Parkserver：

```js
var rpcWorker = require('maus').worker;

//这里我们注册了一个类型为common的Worker
rpcWorker.registerParkserver('http://localhost:8500', 'common', {
    add: (x, y) => x + y,
    fib: fib,
    do: (v, f) => f(v)
})
```

Manager中这样向Parkserver请求Worker：

```js
var rpcManager = require('maus').manager;
var Manager = new rpcManager(8124);

//连接Parkserver
Manager.connectParkserver('http://localhost:8500');

//请求两个类型为common的Worker，注意要附上自己的地址
Manager.getWorker({
    amount: 2,
    workerType: 'common',
    address: 'http://localhost:8124'
}).do(workers => {
    //Do Something...
});
```

Manager可以这样释放Worker：

```js
Manager.end();
```

ps：昨天看完了谷歌那三篇著名的论文（BigTable、GFS、MapReduce），涨了好多姿势，未来这段时间想自己写一个玩具级的分布式文件系统。