title: 开始翻译Bluebird中文文档啦
date: 2015-07-29 22:39:02
categories:
- 技术日记
------

[Bluebird][1]是node.js下的一个异步控制库，基于Promise/A+规范，相比于co、thunks、async，它的语法更加传统，学习成本较低；相比于then.js，它提供的API更加全面，是一个目前各方面都比较优良的promise库。

<!-- more -->
---

#为什么要翻译#
我在使用bluebird的时候，有时会看不懂API说明（很多情况下是因为英文文档中过多地使用了“this/that”这样指代不明的东西，作者毕竟芬兰人），弄得本来很简单的东西却需要花大量时间在阅读理解API上面（以及各种附加的Note）。

而且我发现现在对中文友好且出名的promise库太少，目前只有then.js、thunks有不错的中文文档，然而他们加起来的star也没有bluebird的十分之一（不是我势利眼啊T^T Star数很能看出项目的成熟程度）

中文社区里关于promise的实现有很多文章，但大部分是“介绍”性质的，把promise的一些概念讲完，写一个hello world性质的东西就差不多了，很少提到现有库的实践。翻译这个文档或许对现有状况有帮助吧？

（我才不会告诉你其实真正的原因是暑假除了实习以外，业余时间没啥正事干）


----------
#进度#
具体的进度[在这里][2]

截止到发文章的现在大概翻译了20%左右。。。

目前速度大概是一天3-4个条目，然而身为重度拖延症患者，也是不敢保证绝对速度的，还是看心情吧（逃


  [1]: https://github.com/petkaantonov/bluebird
  [2]: https://github.com/starkwang/bluebird/blob/master/API-CH.md