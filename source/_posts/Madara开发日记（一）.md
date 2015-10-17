title: Madara开发日记(1)
date: 2015-07-11 19:01:56
categories:
- 技术日记
---

自从用了hexo做静态博客之后，就一直想要自己写一个类似hexo的静态博客生成系统

本来一开始想取名叫Jarivs，然而npm上jarvis这个名字已经被占用了，后来想出了madara这个名字/w\，没错，就是宇智波斑的名字（逃

今天花了大半天算是把基础的结构搭好了，有两个部分，madara和madara-cli，分别是核心业务和命令行程序，两个包都上传了npm，直接`npm install madara-cli`就可以了，虽然是个什么功能都没有的0.0.0版本。

https://github.com/starkwang/Madara
https://github.com/starkwang/Madara-cli
<!-- more -->
----------


##Madara-cli##
madara-cli主要处理的是命令行的交互，依赖于核心的madara，本质上只是解析命令行语句然后调用madara核心。

node命令行程序的开发参考了下面这两篇文章：
[七天学会NodeJS][1]
[Node.js 命令行程序开发教程 - 阮一峰的网络日志][2]

madara-cli当前的依赖库主要有
 - commander.js（处理命令行参数）
 - chalk.js（命令行颜色）
 - bluebird（promise/A实现）
 - madara（核心）


----------
##Madara##
这是整个博客工具的核心，上层的API还在设计当中，现在主要实现了“编译全部文章到文章页”的功能。

目前模板引擎选用了jade。


----------


##实现思路##
编译一个静态博客主要有两个部分：

 - HTML
 - CSS/JS/图片静态资源

静态资源的迁移可以直接复制，但可能涉及到url的改变，目前还没有规划。

html整体的编译过程大概是这样：

1、读取源文件，生成一个数据对象，数据对象大概像这样：

    {
        config : {
            //这里是关于文章的一些信息，比如日期、标题、类别
        },
        markdown : "## markdown字符串 ##"
    }

2、渲染html
获取上面的对象数据之后，就可以根据这个对象来渲染html，这里我们用了jade引擎，渲染完毕之后生成一个文件对象，看起来像这样：

    {
        filename : "index",
        data : "<html>...</html>" //html字符串
    }

3、输出
根据上面的文件对象输出文件。


----------
##实现细节##

**1、源文件**
源文件是markdown格式的，但需要在文章头部添加文章的信息（JSON格式），用`---config---`与文章主体相隔，看起来大概是这样：

    {
        "title"    : "文章标题",
        "time"     : "2015-7-11 18:22",
        "tag"      : "tag1,tag2",
        "category" : "日记"
    }
    ---config---
    ##文章正文##
    正文正文正文正文正文正文正文正文正文正文正文正文正文正文

**2、主题编写**
考虑到未来的扩展性，我把所有的模板放在了theme下面，现在暂时的主题叫galaxy（不要问我为什么是这个名字，我只是随便想的/w\），将来可能还会考虑增加更多地主题特性。


----------
##遇到的一些BUG##
**1、异步模块的执行域问题**
异步控制我用的是bluebird，开发中出现了一个执行域的问题，情况大概是这样：

我写了一个函数

    function Foo(){
        var a = [];
        fs.readFileAsync('url','utf-8')
        
            .then(function(data){
                a.push(data);
            })
            
        return a;
    }
结果a是空的。

目前我还没找到这个bug是怎么回事（其实是根本没精力了啊喂），所以我暂时用了同步版本的IO操作，将来再换成异步的。


----------


**2、jade的一个小缺陷**

jade在插入数据的时候会自动把所有字符串转义，似乎不能关闭这个转义功能。导致我用markdown.js生成html字符串插入模板的时候，html字符串被转义了。。。

虽然jade自带了markdown的过滤器，但是这个过滤器只能输入`.md`文件，不能输入一个markdown的字符串，而我却需要这个功能。

所以目前暂时的方法是在模板里插入一个字符串，然后用正则替换成html字符串。

  [1]: http://www.lvtao.net/content/book/node.js.htm
  [2]: http://www.ruanyifeng.com/blog/2015/05/command-line-with-node.html