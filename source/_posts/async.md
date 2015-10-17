title: Promise异步编程下的一些坑
date: 2015-08-07 23:15:47
categories:
- 技术日记
---
最近写madara的时候需要用到大量的promise流程，其中也遇到了很多坑，但与以往不同的是，这些坑大都是反直觉的。所以在这里记录一下。

<!-- more -->
----------
#一、不要用同步的思维来写异步的流程#
在同步的模式下，我们经常能写出差不多长成这样的代码：

    function Foo(){
        var a = 1;
        return a;
    }
    
    var result = Foo();
    
    DoSomething(result);


秉承这个思路，在异步编程的时候容易写出这样的代码（至少我写出来了而且debug了很久/w\）：

    function Foo(){
        var foo;
        somethingIsPromise()
            .then(function(data){
                foo = data;
            });
        return foo;
    }

当我们直接调用它的时候，会发现：

    var a = Foo();  //undefined
    
这个问题是一个典型的**同步思维写异步流程**的反例，在异步流程中，任何异步调用将**直接返回**，而不是等待后续的结果到达后才返回。比如上面的代码可以注释成这样：

    function Foo(){
        var foo;   //声明变量，此时foo为undefined
        
        somethingIsPromise()
            .then(function(data){
                foo = data;
            });
        //上面这句会立即返回，在foo=data之前就已经返回
        
        return foo;   //这个时候foo依然是undefined
    }
    
这意味着什么呢？

在异步编程下（比如使用了promise规范），传统同步编程的很多模式都不再可行。因为传统模式的抽象在本质上是**基于函数的返回值**，而promise下，返回的只是一个promise对象，具体的执行步骤是由一个个promise对象连接在一起组成的**调用链**完成的。

换句话说，这是在同步下面的编程：

    var a = 1;
    var b = Foo1(a);
    var c = Foo2(b);
    doSomethingWith(c);

这是对偶的异步编程：

    Promise(a)
        .then(function(a){
            return Foo1(a);
        }).then(function(b){
            return Foo2(b);
        }).then(function(c){
            return doSomethingWith(c)
        })
        


----------


#二、避免出现promise版的嵌套金字塔#
很多promise的初学者（比如我），即使使用了promise，也非常容易写出像这样的嵌套金字塔：

    Promise(a).then(function(){
        for(var i = 1 ; i < 10 ; i++){
            Promise(b).then(fuunction(){
                // do something
            }).then(function(){
                readFile('file').then(function(){
                    //do something
                })
            })
        }
    })

在`.then()`中再次嵌套`.then()`或者其他promise对象方法并非最佳实践，也是官方不推荐的写法，因为这样的写法实际上和同步编程中的回调地狱没有什么区别。

避免这种问题的需要下面几点：

1、让调用链中的每一步的粒度都尽量的小。
2、调用链中一旦涉及到异步函数（务必也是promise），就返回这个异步函数的promise对象。例如：

    //不好的写法
    Promise(a).then(function(){
        readFile('file').then(function(){
            //do something
        })
    })
    
    //应该这样写
    Promise(a).then(function(){
        return readFile('file')
    }).then(function(data){
        //do something
    })


----------
#三、学会灵活使用promise对象的方法#

Promise对象提供的方法绝不仅仅只有`.then()`和`.catch()`这两种，只调用这两种方法一般是不能解决大多数异步编程问题的。

比如，我们常见的for循环一般是这样的：

    for(var i = 1 ; i < 10 ; i++){
        //do something
    }
很多人会把它结合promise这样用：

    Promise(a).then(function(){
        for(var i = 1 ; i < 10 ; i++){
            Promise(b).then(function(){
                // 这里是一条新的独立的调用链
            })
        }
    })

这样造成的一个最直接的后果就是，这种做法把原来唯一的调用链，通过for循环**分裂成了10条互相独立的调用链**，会对后续的操作带来不必要的麻烦。

解决方法就是使用`.all()`，类似这样：

    Promise(a).then(function(){
        var promiseArray = [];
        for(var i = 1 ; i < 10 ; i++){
            promiseArray.push(Promise(b));
        }
        return promiseArray;
    }).all(promiseArray).then(function(results){
        //do something
    })
    
此外，Promise对象里还有很多类似的方法，在遇到一些可能会写出“不太优雅”的代码的时候，记得一定要看一看文档，找有没有合适的方法调用，而不是死板地只会用`.then()`