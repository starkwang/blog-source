title: Hexo next右侧出现空白问题的修复
date: 2015-07-03 12:37:00
categories: 
- 技术日记
---
最近把原来放在csdn上的博客迁移到了github上，比较了一下最终选择了hexo以及他的一个next主题，比原来csdn乱七八糟土得掉渣的界面好看多了，还兼容移动端。


----------


但是next在移动端出现了一处bug：
<!-- more -->
这是我们正常浏览博文的时候，注意到代码块因为长度，被设置为`overflow: auto;`，即超出长度时出现滚动条：
![此处输入图片的描述][1]


  
  但是当我们向左滑动，变成了这样，出现了右侧的空白区域，整个页面的宽度并不是适配移动端浏览器宽度的：
  ![此处输入图片的描述][2]

后来发现是这个超链接惹的祸，它撑开了整个页面的宽度：
![此处输入图片的描述][3]


----------
##原因##

hexo在把markdown编译成静态页面时，会自动会把下面这样的文本内容（即md中没有声明超链接属性，但是确实是超链接格式的文本）

> http://****

编译成：

    <a href="http://****" target="_blank" rel="external">
        http://****
    </a>

这种编译方式使很多主题（比如next）的编写者，容易忽略掉`<a>`标签中，文本过长的问题，导致这个节点溢出父元素，撑开页面的宽度。


----------
##解决方法##
解决方法其实很简单，需要在全局的base.css中添加

    a { word-wrap: word-break; }
让`<a>`标签在文本超出宽度的时候，自动换行即可。

目前修复这个问题的pull request已经被原作者merge：
![此处输入图片的描述][4]


  [1]: http://img.blog.csdn.net/20150703120745614
  [2]: http://img.blog.csdn.net/20150703120723176
  [3]: http://img.blog.csdn.net/20150703120710937
  [4]: http://img.blog.csdn.net/20150703122718763