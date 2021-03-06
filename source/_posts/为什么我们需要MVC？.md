title: 为什么我们需要MVC？
date: 2015-05-15 11:53:02
categories:
- 技术日记
------
作为一只前端汪，或许经常能听到MVC这个东西，即模型（Model）、视图（View）、控制器（Controller），很多人（包括我）或许在最初都觉得这个东西虽然解耦非常清晰，能让代码结构很明了，扩展性也很好，但是不能理解这个东西为何会跟“画网页”的前端扯上关系。

<!-- more -->
----------
## MVC为何好用？ ##

最近百度IFE的任务也慢慢做到了第三个，[第三个任务](https://github.com/baidu-ife/ife/tree/master/task/task0003)是用localStorage实现一个本地的“日程笔记”，一开始我觉得这个东西并不难嘛，只是DOM交互多一点而已，所以把重点放到了实现数据储存上面，设计了一个符合第三范式的数据库，而没有把重点放到模块化上面。

数据库结构很快弄完了，然后就是喜闻乐见的“画页面”，这个应用有大量的按钮、弹出菜单、DOM操作，但是天真的我觉得这些东西以前都写过嘛，并不需要上MVC这种烦死人的东西嘛。然后就天真地开始码码码了。


![这里写图片描述](http://img.blog.csdn.net/20150515114656405)


对于弹出框，我在页面里放了一个DIV，display为none，然后当点击“添加xxx”时会改写display让它显示出来，这是一种非常简单的弹出框设计。

![这里写图片描述](http://img.blog.csdn.net/20150515115238498)

但是点击确定保存数据之后，我会把修改本地的数据模型，并且存到LocalStorage里面，但是遇到了一个问题：

**添加新的东西之后，如何让视图自动进入新的 分类/子分类/任务 里？**

由于没有使用MVC，我只能通过模拟几次点击事件的方式来修改视图，这种方法非常傻，因为整个数据结构有三层，如果用户添加了一个最底层的“任务”，就会导致我需要模拟三次点击；添加了第二层的“子分类”，我需要模拟两次……

这样就导致了代码的冗余，本来应该很单纯的数据储存，我却要把精力放在状态转换上面，但是状态转化却只能靠“模拟点击”这种傻傻的方法实现。


----------


**问题的本质**

这个问题其实并不是一个极其罕见的问题，因为它本质上就是一个“状态转换、保存”以及“状态改变之后的渲染”问题，也就是我们MVC框架中控制器（Controller）和视图（View）的任务，但是我这里没有设计MVC模式，导致出现了dirty的解决方式。

在这个任务里，我本质上其实是把状态放到了DOM里面，然后获取DOM，改变DOM，这样状态才得以改变和保存。这导致了状态和视图实际上是混合在一起的，非常不利于解耦，页面中任何事件涉及到状态改变时，我也需要写相应的视图改变的代码，让我的代码看起来像这样：

```
if(事件发生){
	/*
	一大堆代码改变数据结构
	*/


	/*
	一大堆代码模拟点击，改变当前页面的状态以及视图，又臭又长，非常不优雅
	*/
}
```

但是在MVC里，我们需要的只是一个视图渲染器、一个状态控制器而已，用户的任何操作，改变的是“页面状态”，这个状态或许是一个Object或者其他的数据类型，然后渲染器根据这个“状态”去改变DOM（一般情况下只是改变局部页面），这时代码就变成了这样：

```
if(事件发生){
	/*
	一大堆代码改变数据结构、状态模型
	*/
	
	render();   //渲染器函数
}
```
当然还需要一个渲染器：

```
function render(status){
	//一些代码
}
```
这样的写法让状态和视图完全分离开，当事件发生时，过程是这样的：

 1. 事件发生
 2. 改变状态模型
 3. 根据状态模型，渲染视图

由于“状态模型”的存在，让我们不需要把状态放在DOM中，避免了大量的性能低下而且不优雅的DOM查找以及字符串的处理，不仅让代码结构清晰，还可以提升性能。

----------
## 什么情况下适用MVC ##
首先我们要知道，MVC并不是万能的，它不会适用于全部的场景，那么它适用于什么情况呢？

如果你需要在前端上实现一个涉及到状态转换、多重数据交互等一系列功能的应用（例如web版的印象笔记、任何在线邮箱），那么MVC或许适合你。

如果你只是需要一个展示性质的网页（例如大多数的新闻网站），不会有太多的事件、DOM转变，这个时候用使用MVC或许反而加大了工作量以及复杂程度。


