title: 所以来说说新的复旦选课助手吧
date: 2015-10-17 23:17:13
tags: 
- 技术日记
---

啊真的好久好久没有写过东西了，今天发现上一篇文章还是在暑假实习的时候写的。

这段时间（8月底到现在10月中旬）做的事情还蛮多。

先弄了个写着玩的基于canvas的图像处理的库[Simage][1]；

还有个在console或者terminal里用来输出字符图画的小玩具[alphabetJS][2]；

给学生网内部培训写了个基于websocket的小黑板[Blackboard][3]（然而拖延症晚期一直没有补上readme），不过如果连接复旦内网的话，访问http://stu.fudan.edu.cn/blackboard 就可以看到啦。你输入任何字符，所有打开这个网页的浏览器都可以看到（当然包括其他人的设备）。

当然最重要的还是[新版复旦选课助手][4]，网址是http://stu.fudan.edu.cn/xk ，源码在[github.com/starkwang/XKHelper][5]。

现在还只是测试版本，课程的数据只导入了部分并且是这个学期的，不过功能都已经实现了，将来会考虑接入UIS认证以及搭一个可以评价课程的课程论坛，这样就可以抛弃老旧的GPA TOOLS啦。

<!-- more -->
-------

#文件结构
这是一个基于构建angular的单页应用（后台生成二维码部分是用node写的），源码放在src目录下。

#src/lib
这里放的是主要的JS库，依赖如下：

 - angular、angular-route、angular-animate
 - [ngClipBoard][6]（我自己用clipboard.js封装的一个directive）
 - csv.js
 - fastclick
 - jquery、jqeury-qrcode


#src/js
主要的angular逻辑代码，具体的说明如下：

***app.js***
入口文件，依赖注入、路由都在这里

***baseService.js***
抽象出的服务层，具体有：

 - 课表、收藏夹的模型
 - 网络服务
 - 搜索
 - localStorage的管理
 - 以及一些可复用的方法（例如time parser）


***XXXXController.js***
各模块对应的controller，对应关系如下：
allController   所有课程 
collectionController   收藏夹      
courseTableController   课表      
courseTotalController   课程清单、考试时间      
sidebarController   侧边栏      
searchController   搜索      


----------

#构建

构建工具我选择了gulp（话说下一个项目我想试试webpack+react的组件化），使用sass编写样式（作为css的超集写起来挺爽的），具体的构建流程在README中有。

---------
#一些实现细节

**1、课程数据**
课程数据是本地化的，具体的数据放在src/lib/data.js中，数据以CSV字符串的格式读入，用csv.js解析成全局对象COURSE_DATA存在内存中（因为一共就3000多门课，数据量比较小）。

**2、搜索**
搜索功能封装在baseService中，由于COURSE_DATA只是普通的JS对象，所以每次搜索都需要遍历，并没有做索引/w\

`baseService.search(specification)`方法会遍历每门课程，判定是否与条件匹配。判定是否匹配的方法在BaseService:188行，`matchCourse(specification, course)`

**3、controller之间的通信**
这个应用中有几处需要跨controller通信的地方，比如当你在搜索栏中加入一门课时，课表中需要做出相应的反应，而这两处是不同的controller。
所以定义了以下事件：

 - courseModelUpdate   课表模型变化
 - collectionUpdate  收藏夹模型变化
 - showSearch    弹出搜索条

--------------
#存在的问题
1、现在对于COURSE_DATA的结构还严重依赖于源数据的结构，我并没有去写一个parser把源数据转换为某种标准结构。换句话说，如果教务处给的CSV格式有变化，我这边相对应的服务层也要变化，不过这个问题不太严重。


2、在IE甚至EDGE下，angular的渲染性能极差，导致搜索结果过多时会出现明显的卡顿甚至假死。搜索这一块未来会考虑移到服务器端用node写（现有的方法都是可以在node里复用的）。

3、我没有使用诸如Webpack、Browserify之类的打包工具，导致没有项目构建这个过程，在开发中有些不便（代码都是压缩后的，很难溯源）。

  [1]: https://github.com/starkwang/Simage
  [2]: https://github.com/starkwang/alphabetJS
  [3]: https://github.com/starkwang/Blackboard
  [4]: http://stu.fudan.edu.cn/xk/
  [5]: https://github.com/starkwang/XKHelper
  [6]: https://github.com/starkwang/ngClipBoard