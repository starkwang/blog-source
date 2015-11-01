title: Webpack + Angular的组件化实践
date: 2015-10-27 16:54:13
tags:
- 技术日记
---
最近写复旦二手平台的时候开始尝试用一直推崇了很久的组件化。经过一番抉择之后选择了 webpack + angular 的组合。所以在这里分享一下具体的实践流程。
<!-- more -->
-------
#Webpack
Webpack是目前比较流行的前端打包工具，它同时支持AMD、CMD两种模块写法，也原生支持npm或者bower安装的模块。它还能给css、scss、less、coffeescript、es6、图片、html以及诸如jade、ejs的模板打包。

**所以有什么卵用呢？**

简单地说就是，

1、原来你需要在`<script>`中引入angular或者其他的npm模块（有些npm模块甚至没有提供可以直接在浏览器端引用的js文件），现在只需要：

```
npm install angular
```

然后在app.js中：

```js
angular = require('angular');
var app = angular.module('myApp',[]);
```
然后执行

```
webpack app.js bundle.js
```
webpack会自动分析依赖，然后编译，这样`bundle.js`就是你想要的东西了。


2、组件化的时候你要在页面中引入一大堆东西，比如这样：

```html
<!--index.html-->
<script type="text/javascript" src="header.js">
<script type="text/javascript" src="tab.js">
<script type="text/javascript" src="waterfoo.js">
<link rel="stylesheet" type="text/css" href="header.css">
<link rel="stylesheet" type="text/css" href="tab.css">
<link rel="stylesheet" type="text/css" href="waterfoo.css">

```
当然你可能不会用如此傻的方式引入组件，但如果用了webpack之后只需要这样：


```js
//index-bundle.js
require('header.js');
require('header.scss');
require('tab.js');
require('tab.scss');
require('waterfoo.js');
require('waterfoo.scss');
```
然后在index.html中引入打包好的js即可（没错连scss都给你打包好了/w\ 它甚至还能把图片打包成base64，然后替换所有url）：

```html
<!--index.html-->
<script type="text/javascript" src="bundle.js">
```

上面只是最基础的用法，所以有些人可能要吐槽这不是browserify也能干么/w\ 可以看看这篇文章[《用webpack来取代browserify》
](http://segmentfault.com/a/1190000002490637)，以及webpack还有很多酷炫的功能这里就不再赘述。

------
#项目实践流程

回到正题吧，复旦二手平台这个项目更像是多页面的网站而不是单页面的web应用，所以我决定尝试用node去写一个渲染层（个人感觉Koajs目前还不太成熟，所以选用了Express4.0），这样后台写JAVA的同学就只需要给我提供数据API就行了，把数据库接口化。

渲染引擎用的是jade，而angular的角色更像是一个页面的“控制器”，控制由jade生成的页面，而不是去自行加载html自行渲染（说白了就是放弃了用angular渲染页面）。

##Webpack和Angular的结合

Angular自带了Module以及Directive机制，但Angular1.x版本下，我觉得这些机制不太适合做这种多页面网站的组件化，而且也违背了选用jade渲染的初衷。

Angular自己有自己独特的依赖注入以及模块声明方式，看起来似乎和Webpack是水火不容的，但事实上他们完全可以融合。只需要多几行代码：

主文件app.js大概长这样：

```js
var angular = require('angular');
var starkAPP = angular.module('starkAPP', [
]);
module.exports = starkAPP;

```
注意到我们在这里把starkAPP作为模块的接口暴露出去，然后我们就可以这样写controller：

```js
//someController.js
var starkAPP = require('./app.js');
starkAPP.controller('someController', ['$scope', function($scope) {
    //...
}])
```
运行一下`webpack someController.js bundle.js `即生成了一个可以使用的bundle.js。

当然如果你有一堆controller、directive、service，最好用个`main.js`全部声明一下依赖：

```js
//main.js
require('./Controller1');
require('./Controller2');
require('./Controller3');
require('./Service');
require('./Directive');
```
##目录结构设计
这里我只放了浏览器端的文件结构，整个的项目结构可以看[这里](https://github.com/starkwang/FDSHM)

```
|package.json 存放npm相关的配置

|gulpfile.js gulp的配置文件

|webpack.config.js 存放webpack相关的配置

|build 存放构建完毕的资源文件

|node_modules 不解释了= =

|src 源代码

    └── components 组件
    
    	 ├── angular angular组件，比如各种directive、service
    
    	 ├── base 需要全站引入的组件，比如reset.css
    
    	 └── header 头部组件
    	 
    	 	 ├── header.jade

			 ├── header.scss
			 
			 └── header.js
			 
	 └── pages 页面定义文件
	 
		 └──  index 首页配置文件
		 
		 	 ├── index.js
		 	 
		 	 └── index.scss
		 	 
	 └── template 提供给node渲染的jade模板
	 
	 	 └── index.jade 首页模板
```

看文件结构绝对是云里雾里的，下面详细说明：

**1、首先这是首页的模板`index.jade`**

```jade
html(ng-app="starkAPP")
    head
        link(rel='stylesheet',href='/static/css/index.css')
        script(type='text/javascript',src='/static/js/index.bundle.js')
    body(ng-view)
        include ../components/header/header.jade
```

注意到我们引入了header的jade，以及两个文件`index.css`和`index.bundle.js`

**2、`index.css`是啥？**

它是`pages/index/`里面的`index.scss`编译成的：

```scss
// pages/index/index.scss
@import '../../components/header/header';
@import '../../components/base/base';
```
注意到我们在这里引入了header.scss

**3、`index.bundle.js`是啥？**

它是`pages/index/`里面的`index.js`经过webpack打包成的东西

```js
// pages/index/index/js
require('../../components/angular/app.js');
require('../../components/header/header.js');

```
我们在这里引入了angular以及`header.js`

**总之，pages下面放的就是各个页面的组件依赖文件**

比如我的首页index依赖了一个组件header，那么我需要在`index.js`和`index.scss`中声明我依赖了`header.js`以及`header.scss`

其实用webpack打包的话，只需要一个定义文件就可以同时打包js和scss，但我还不太确定webpack打包scss这种方法是否成熟。

##项目构建


自动构建工具我选择了gulp全家桶，简单地说就是读取`src/pages/[page-name]`下面所有的js、scss文件，把他们编译到对应的`build/[page-name]`下面，并且监听文件变化以便热替换。

这是我的`gulpfile.js`:

```js
var gulp = require('gulp');
var sass = require('gulp-sass'),
    autoprefixer = require('gulp-autoprefixer'),
    minifycss = require('gulp-minify-css'),
    uglify = require('gulp-uglify'),
    clean = require('gulp-clean'),
    webpack = require('gulp-webpack');
var webpackConfig = require('./webpack.config');
gulp.task('default', ['clean', 'watch', 'sass:watch', 'sass', 'webpack']);

gulp.task('sass:watch', function() {
    gulp.watch('src/pages/*/*.scss', ['sass']);
    gulp.watch('src/components/*/*.scss', ['sass']);
});
gulp.task('sass', function() {
    gulp.src(['src/pages/**/*.scss'])
        .pipe(sass.sync().on('error', sass.logError))
        .pipe(minifycss())
        .pipe(gulp.dest('build/pages'));
});
gulp.task('clean', function() {
    return gulp.src(['build/pages'], {
            read: false
        })
        .pipe(clean());
});
gulp.task("webpack", function() {
    return gulp
        .src('./')
        .pipe(webpack(webpackConfig))
        //.pipe(uglify())
        .pipe(gulp.dest('./build/pages'));
});
gulp.task('watch', function() {
    gulp.watch('src/components/*/*.js', ['webpack']);
    gulp.watch('src/pages/*/*.js', ['webpack']);
});

```
以及简化后的`webpack.config.js`：

```
module.exports = {
    entry: {
        'index/index': './src/pages/index/index.js'
    },
    output: {
        filename: '[name].bundle.js'
    }
};

```

##怎么写一个组件？
**比如现在我们要写一个waterfoo（瀑布流）组件**

首先我们在`src/components/`下面新建一个文件夹`waterfoo `，然后建立

 - waterfoo.jade
 - waterfoo.scss
 - waterfoo.js

 
分别对应waterfoo组件的模板、样式、以及行为，当然waterfoo组件完全可以依赖其他更低层级组件，只需要在相应的文件中声明依赖即可。

##怎么在页面中加入组件？
**比如现在我们要把waterfoo（瀑布流）组件加到首页index中**

首先在`src/template/index.jade`中引入模板：

```jade
include ../components/waterfoo/waterfoo.jade
```

然后在`src/pages/index/`下面的`index.js`、`index.scss`配置依赖：

```js
//index.js
require('../../components/waterfoo/waterfoo');
```
```scss
//index.scss
@import '../../components/waterfoo/waterfoo';
```
##有更优雅更傻瓜的方法吗？

当然有，未来的期望是用webpack把js和scss一起打包，并且把`template`和`pages`文件夹合并（具体配置express渲染路径的方法我还在探索），大概就是这样的效果：

`src/pages/index/`下面放着首页的配置文件：

 - index.jade
 - index-config.js
 - index.scss （一些非组件的样式）

**index.jade**是模板

 ```jade
 html(ng-app="starkAPP")
    head
        script(type='text/javascript',src='/static/js/index.bundle.js')
    body(ng-view)
        include ../components/header/header.jade
 ```
 
 **index-config.js**声明依赖的组件
 
 ```js
require('../../components/angular/app.js');
require('../../components/header/header.js');
require('../../components/header/header.scss');
require('../../components/waterfoo/waterfoo.js');
require('../../components/waterfoo/waterfoo.scss');
require('./index.scss');
 ```
 -------
 
#最后

这个项目的源码在[Github · starkwang/FDSHM](https://github.com/starkwang/FDSHM)，FDSHM是Fudan Secondhand Market的缩写/w\，现在还在慢慢地写组件中，争取这学期上线吧……（缺人啊QAQ）