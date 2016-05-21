title: 手写一个CommonJS打包工具（二）
date: 2016-04-25 20:42:19
tags:
- 技术日记
---
前一篇文章：[手写一个CommonJS打包工具（一）](http://zhuanlan.zhihu.com/p/20731484)

上周就写完了这个小东西：

[https://github.com/starkwang/tiny-commonjs-pack](https://github.com/starkwang/tiny-commonjs-pack)

<!-- more -->

一直很忙（lǎn）没有补readme和注释，今天补上了。目前还不支持node_modules和global变量，有时间再加吧。

其实具体的实现蛮简单的，所以就不再一段段解释，直接写到代码的注释里了：

[tiny-commonjs-pack/commonjs-pack.js at master · starkwang/tiny-commonjs-pack · GitHub](https://github.com/starkwang/tiny-commonjs-pack/blob/master/commonjs-pack.js)

懒得点开的话可以直接看下面：

```js
import Promise from "bluebird";
import fs_origin from "fs";
import { js_beautify } from "js-beautify";
import path from "path";

var fs = Promise.promisifyAll(fs_origin);

//__MODULES用于映射moduleID
var __MODULES = [
    // 0: 'index',
    // 1: 'module1'
    // 2: 'test/module2'
];

//读取命令行参数
if (process.argv[2]) {
    console.log("starting bundle " + process.argv[2]);
    pack(process.argv[2]);
} else {
    console.log("No File Input");
}


function pack(fileName) {
    var name = fileName.replace(/\.js/, "");

    //基本的module模板
    var str = "function(module, exports, require, global){\n{{moduleContent}}\n},\n";

    //递归打包
    bundleModule(name, './')
        .then(() => {
            console.log(__MODULES);

            //把模块名替换成数字ID
            return Promise.map(__MODULES, (moduleName => replaceRequireWithID(moduleName)))
        })
        .then(moduleContents => {
            //合并模块
            var modules = "[";
            moduleContents.forEach(content => {
                modules += str.replace(/{{moduleContent}}/, content);
            })
            return modules += "]"
        })
        //输出
        .then(modules => fs.readFileAsync("packSource.js", "utf-8").then(content => content + "(" + modules + ")"))
        .then(result => js_beautify(result))
        .then(x => log(x))
        .then(result => fs.writeFileAsync("bundle.js", result))
        .then(() => console.log("bundle success!"));
}


//递归打包的方法
//接收两个参数：moduleName是模块名，nowPath是当前路径
function bundleModule(moduleName, nowPath) {
    console.log("reading :", path.normalize(nowPath + moduleName + '.js'));
    return fs.readFileAsync(path.normalize(nowPath + moduleName + '.js'), 'utf-8')
        .then(contents => {
            //在__MODULES中注册这个模块名
            __MODULES.push(path.normalize(nowPath + moduleName))
            return contents;
        })
        .then(contents => matchRequire(contents))//解析出require
        .then(requires => {
            if (requires.length > 0) {
                //对每个require分别递归打包
                return Promise.map(requires, (requireName => {
                    return bundleModule(requireName, path.dirname(nowPath + moduleName) + "/")
                }))
            } else {
                return Promise.resolve();
            }
        })
}

//把模块名替换成ID的方法
//接收一个参数：moduleName即模块名
function replaceRequireWithID(moduleName) {
    var dirPath = path.dirname(moduleName) + '/';
    return fs.readFileAsync(moduleName + '.js', 'utf-8')
        .then(code => {
            matchRequire(code).forEach(item => {
                var reg1 = new RegExp("require\\(\"" + item + "\"\\)");
                var reg2 = new RegExp("require\\(\'" + item + "\'\\)");
                var modulePath = path.normalize(dirPath + item);
                var moduleID = __MODULES.indexOf(modulePath);
                code = code.replace(reg1, "require(" + moduleID + ")").replace(reg2, "require(" + moduleID + ")");
            })
            return code;
        })
}


//解析依赖的模块名
function matchRequire(code) {
    var requires1 = code.match(/require\("\S*"\)/g) || [];
    var requires2 = code.match(/require\('\S*'\)/g) || [];
    return requires1.map(item => item.match(/"\S*"/)[0]).map(item => item.substring(1, item.length - 1))
        .concat(requires2.map(item => item.match(/'\S*'/)[0]).map(item => item.substring(1, item.length - 1)));
}


function log(a) {
    console.log(a);
    return a;
}
```