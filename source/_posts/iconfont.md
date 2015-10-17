title: iconfont的最佳实践
date: 2015-07-14 18:58:53
categories:
- 技术日记
--------------

iconfont可以说是前端神器之一，超小的体积，矢量图的效果，还可以随意更改颜色、大小、粗细，更可怕的是竟然还兼容IE6。真是旅行居家馈赠亲友殴打产品经理（误）的利器啊！

这几天一直在做一个很繁琐的任务，要把堆糖主站[www.duitang.com][1]及其子页面所有图片icon迁移成iconfont，所以这里涉及到了一个问题：

**怎样才能优雅地使用iconfont？**
<!-- more -->

----------
##一、直接编码##
在阿里的[iconfont库][2]上，它告诉你，你应该这样用iconfont：
1、font-face声明字体

    @font-face {font-family: 'iconfont';
        src: url('iconfont.eot'); /* IE9*/
        src: url('iconfont.eot?#iefix') format('embedded-opentype'), /* IE6-IE8 */
        url('iconfont.woff') format('woff'), /* chrome、firefox */
        url('iconfont.ttf') format('truetype'), /* chrome、firefox、opera、Safari, Android, iOS 4.2+*/
        url('iconfont.svg#iconfont') format('svg'); /* iOS 4.1- */
}

2、定义使用iconfont的样式

    .iconfont{
        font-family:"iconfont";
        font-size:16px;font-style:normal;
    }
3、挑选相应图标并获取字体编码，应用于页面

    <i class="iconfont">&#33</i>

如果只有少量的iconfont，这样做确实没有什么问题，但是在工程上我们遇到的情况往往是页面中含有大量的iconfont，这就涉及到一个语义化的问题：

**我们要怎么知道`&#33`这东西究竟是个啥样的icon啊？**
**(⊙o⊙)…**


---------
##二、使用:before或者:after伪类##
在html中，语义化的标签是很重要的，诸如`<i class="iconfont">&#33</i>`这样的标签是没有任何语义的，只能知道这是一个iconfont而已。

所以我们可以用CSS3中，`:before`或者`:after`伪类来为我们联系起元素和icon，比如我们的CSS可以这样写：

    .icon-mainlogo:before{
        content : "&#33";    
    }
    
然后就可以优雅地建立iconfont，而且是带语义的：

    <i class="iconfont icon-mainlogo"></i>
    
但如果事情是这么简单的话就好了，现在又跑出来一个问题：

**IE6、IE7是不支持`:before`和`:after`伪类的，这样写会导致IE6/7下面iconfont消失。**

对于这个新的问题我们只有两条路：

 1. 针对IE做hack
 2. 再换一种方法

----------
##三(1)、针对IE的hack##

对于IE不支持伪类的问题，可以参考stackoverflow的这个问题：
http://stackoverflow.com/questions/4181884/after-and-before-css-pseudo-elements-hack-for-ie-7

总之有两种解决方法:

 1. 使用IE8.js，这是一个谷歌提供的让IE6/7的HTML、CSS行为符合W3C标准的js脚本（怎么有一种黑微软的感觉= =）
 2. 使用一个jQuery插件jQuery Pseudo（其实原理和上面差不多）

但在实际的工程运用中，考虑到安全性、可维护性的问题，非必要情况下应该要尽量避免使用第三方库（虽然是谷歌提供的），所以这个方法或许更适合个人开发者。

后来我尝试了一种新的hack方法：**针对IE6/7用js脚本插入iconfont字符。**

例如这样：

    if(isIE6 || isIE7){
        $('.icon-mainlogo').html('&#33');
    }

但这样依然有个问题，这个js脚本只在dom建立后执行一次，但是在一些情形下，dom是后期建立、更改的。

如果我们在不定的时刻给dom添加了一个iconfont，我们很难监听到这些事件的发生然后插入字符（可以做到但是成本太高），所以这种方法也不是太好。

----------
##三(2)、最后的妥协##

回到问题自身上，其实它本质上就是:

如何在**保证语义化**的情况下**兼容IE6/7**地使用iconfont？

只有iconfont字符串，我们单凭代码就无法知道对应的icon；添加class然后插入字符串，兼容性就会出现问题。

所以现在暂时的妥协方法是，在html中两种方法都使用，即直接使用iconfont字符串的同时，加一个class作为标记，使之具有语义。

    <i class="iconfont icon-mainlogo">&#33</i>

 所以写了这么多，最后的结论就是……

 对于iconfont的最佳实践还依然在探索中（举着锅盖逃）
  [1]: http://www.duitang.com
  [2]: http://iconfont.cn