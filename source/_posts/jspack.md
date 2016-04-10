title: 手写一个CommonJS打包工具（一）
date: 2016-04-10 20:42:19
tags:
- 技术日记
---
CommonJS 是一个流行的前端模块化规范，也是目前 NodeJS 以及其模块托管仓库 npm 使用的规范，但目前暂无浏览器支持 CommonJS 。要想让浏览器用上这些模块，必须转换格式。

这个系列的文章，我们会一步步完成一个基于 CommonJS 的打包工具，类似于一个简单版的 Browserify 或者 Webpack 。

<!-- more -->
------
#一、原理

与 NodeJS 环境不同，浏览器中不支持 CommonJS 的主要原因是缺少了以下几个环境变量：

- module
- exports
- require
- global

换句话说，打包器的原理就是模拟这四个变量的行为。

比如我们有一个`index.js`文件，依赖了`module1`和`module2`两个模块，并且`module1`依赖`module2`：


```js
//index.js
var module1 = require("./module1");
var module2 = require("./module2");

module1.foo();
module2.foo();

function hello(){
    console.log("Hello!");
}

module.exports = hello;
```

```js
//module1.js
var module2 = require(module2);
console.log("initialize module1");

console.log("this is module2.foo() in module1:");
module2.foo();
console.log("\n")

module.exports = {
    foo: function(){
        console.log("module1 foo !!!");
    }
};
```
```js
//module2.js
console.log("initialize module2");
module.exports = {
    foo: function(){
        console.log("module2 foo !!!");
    }
};
```
把它放入一个匿名函数内，通过这个匿名函数注入 `require`、`modules`、`export`、`global`变量（我们暂时不实现global）

```js
function(module, exports, require, global){
    var module1 = require("./module1");
    var module2 = require("./module2");

    module1.foo();
    module2.foo();

    function hello(){
        console.log("Hello!");
    }
    
    module.exports = hello;
}
```

现在我们用一个 `modules` 对象来存入这些匿名函数：

```js
//modules
{
    "entry": function(module, exports, require, global){
        //index.js
        var module1 = require("./module1");
        var module2 = require("./module2");
        module1.foo();
        module2.foo();
        function hello(){
            console.log("Hello!");
        }
        module.exports = hello;
    },
    "./module1": function(module, exports, require, global){
        var module2 = require("./module2");
        console.log("initialize module1");

        console.log("this is module2.foo() in module1:");
        module2.foo();
        console.log("\n")

        module.exports = {
            foo: function(){
                console.log("module1 foo !!!");
            }
        };
    },
    "./module2": function(module, exports, require, global){
        console.log("initialize module2");
        module.exports = {
            foo: function(){
                console.log("module2 foo !!!");
            }
        };
    }
}
```
下面我们实现一个简单的 `require` 函数：

```js
//这个对象用于储存已导入的模块
var installedModules = {};

function require(moduleName) {
    //如果模块已经导入，那么直接返回它的exports
    if(installedModules[moduleName]){
        return installedModules[moduleName].exports;
    }
    //模块初始化
    var module = installedModules[moduleName] = {
        exports: {},
        name: moduleName,
        loaded: false
    };
    //执行模块内部的代码，这里的 modules 变量即为我们在上面写好的 modules 对象
    modules[moduleName].call(module.exports, module, module.exports,require);
    //模块导入完成
    module.loaded = true;
    //将模块的exports返回
    return module.exports;
}
```

最后只要把我们上面写好的 `modules` 对象以立即执行函数的形式传入这个 `require` 函数就可以了，以下是完整的代码：

```js
(function(modules){
    //这个对象用于储存已导入的模块
    var installedModules = {};
    function require(moduleName) {
        //如果模块已经导入，那么直接返回它的exports
        if(installedModules[moduleName]){
            return installedModules[moduleName].exports;
        }
        //模块初始化
        var module = installedModules[moduleName] = {
            exports: {},
            name: moduleName,
            loaded: false
        };
        //执行模块内部的代码，这里的 modules 变量即为我们在上面写好的 modules 对象
        modules[moduleName].call(module.exports, module,        module.exports,require);
        //模块导入完成
        module.loaded = true;
        //将模块的exports返回
        return module.exports;
    }
    //入口函数
    return require("entry");
})({
    "entry": function(module, exports, require, global){
        //index.js
        var module1 = require("./module1");
        var module2 = require("./module2");
        module1.foo();
        module2.foo();
        function hello(){
            console.log("Hello!");
        }
        module.exports = hello;
    },
    "./module1": function(module, exports, require, global){
        var module2 = require("./module2");
        console.log("initialize module1");

        console.log("this is module2.foo() in module1:");
        module2.foo();
        console.log("\n")

        module.exports = {
            foo: function(){
                console.log("module1 foo !!!");
            }
        };
    },
    "./module2": function(module, exports, require, global){
        console.log("initialize module2");
        module.exports = {
            foo: function(){
                console.log("module2 foo !!!");
            }
        };
    }
});
```

事实上，我们短短的这几十行代码模仿了 Webpack 的部分实现。但我们依然在使用诸如 `"./module1"` 这样的字符串作为模块的唯一识别码，这是一个明显的缺陷，存在多层级文件时，这个名称很容易冲突。

在 Browserify 或 Webpack 这样的生产级工具里，一般使用数字作为函数的唯一识别码，例如它可能会把（以 Webpack 为例）：

```js
var module1 = require("./module1");
```

编译成：

```js
var module1 = __webpack_require__(1);

```

------
#二、小结
我们在这里实现了一个最简单的 CommonJS 标准的执行器，接下来的文章中我们会做以下事情：

1、实现 global 变量

2、用 moduleID 替代 moduleName

3、写一个命令行小工具

4、支持 node_modules 和多层级文件
