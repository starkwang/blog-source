title: Javascript元编程（一）
date: 2016-04-16 13:42:56
categories:
- 技术日记
------

这几天把一年多前买的《松本行弘的程序世界》重新看了看，很多当时不能理解的东西现在再去看真是茅塞顿开呀，看到元编程那一段真是把我震撼到了，后来发现 Javascript 里其实也是有一些支持元编程的特性的，今天就用一个 DEMO 示范一下吧。

<!-- more -->
------
#什么元编程

“元编程”这个名字看起来高端大气上档次，它的含义也是相当高端：“__写一段自动写程序的程序__”，不要误会，我们做的可不是人工智能。

言简意赅地说，元编程就是将代码视作数据，直接用字符串 or AST or 其他任何形式去操纵代码，以此获得一些维护性、效率上的好处。

Javascript 中，`eval`、`new Function()`便是两个可以用来进行元编程的特性。

------
#原始示例

现在我们有一堆用户的数据，具体字段有`name`,`sex`,`age`,`address`等等，通过类似 `/get_name?id=123456` 来拉取数据

那么我们很容易写出这样的代码：

```js
class User {
    constructor(userID) {
        this.id = userID;
    }

    get_name() {
        return $.ajax(`/get_name?id=${this.id}`);
    }

    get_sex() {
        return $.ajax(`/get_sex?id=${this.id}`);
    }

    //下面是get_age、get_address......
}
```
这段代码的问题在哪呢？

首先，用户数据有多少个字段，我们就要定义多少个 `get_something` 方法，更可怕的是这些方法里逻辑都是重复的，都是一个简单的 ajax。

#进阶（一）

我们可以把拉取数据的逻辑封装到 `__fetchData` 里：

```js
class User {
    constructor(userID) {
        this.id = userID;
    }
	
	__fetchData(key) {
        //这是一个private方法，直接调用类似__fetchData("age")是不被允许的
        return $.ajax(`/get_${key}?id=${this.id}`)
    }

    get_name() {
        return this.__fetchData('name');
    }

    get_sex() {
        return this.__fetchData("sex");
    }

    //下面是get_age、get_address......
}
```

然后，冗余的问题可以通过`registerProperties`来解决：

```js
class User {
    constructor(userID) {
        this.id = userID;
        this.registerProperties(["name", "age", "sex", "address"]);
    }

    registerProperties(keyArray) {
        keyArray.forEach(key => {
            this[`get_${key}`] = () => this.__fetchData(key);
        })
    }

    __fetchData(key) {
        //这是一个private方法，直接调用类似__fetchData("age")是不被允许的
        return $.ajax(`/get_${key}?id=${this.id}`)
    }
}
```

#进阶（三）
到目前为止我们都没有涉及到任何元编程的概念，下面我们加上更高的需求：

在拉去数据之后，我们要对部分数据进行一定的处理，比如对 `name` 我们要去掉首尾的空格，对 `age` 我们要加上一个 `岁` 字。具体的处理方法定义在 `__handle_something` 里面。

这里我们便可以通过 `new Function()` 来动态生成函数，元编程开始显现威力：

```js
class User {
    constructor(userID) {
        this.id = userID;
        this.registerProperties(["name", "age", "sex", "address"]);
    }

    registerProperties(keyArray) {
        keyArray.forEach(key => {
            //注意这里的fnBody内部依然采用ES5的写法，因为babel目前不会编译函数字符串。
            var fnBody = `return this.__fetchData("/get_${key}?id=${this.id}")
                    .then(function(data){
                        return this.__handle_${key}?_this.handle_${key}(data):data;
                    })`;
            this[`get_${key}`] = new Function(fnBody);
        })
    }

    __handle_name(name) {
        //do somthing with name...
        return name;
    }

    __handle_age(age) {
        //do somthing with age...
        return age;
    }

    __fetchData(key) {
        //这是一个private方法，直接调用类似__fetchData("age")是不被允许的
        return $.ajax(`/get_${key}?id=${this.id}`)
    }
}
```

------
#进阶（四）

下面我们让需求更加变态一点：

1. 数据并非通过 ajax 直接拉取，而是通过一个别人封装好的 `UserDataBase` 里的方法来拉取；
2. 数据的字段并非只有`name`,`sex`,`age`,`address`四个，而是要根据 `UserDataBase` 里给你的方法决定。给你1000个get不同字段的方法，User类里也要有对应的1000个方法。

```js
class UserDataBase {
    constructor() {}
    get_name(id) {}
    get_age(id) {}
    get_address(id) {}
    get_sex(id) {}
    get_anything_else1(id) {}
    get_anything_else2(id) {}
    get_anything_else3(id) {}
    get_anything_else4(id) {}
    //......
}
```

这里我们就需要用到 JS 的反射机制来读取所有拉取字段的方法，然后通过元编程的方式来动态生成对应的方法。

```js
class User {
    constructor(userID, dataBase) {
        this.id = userID;
        this.__dataBase = dataBase;
        for (var method in dataBase) {
            //对每一个方法
            this.registerMethod(method);
        }
    }

    registerMethod(methodName) {
        //这里除去了前置的"get_"
        var propertyName = methodName.slice(4);
        
        //注意这里拉取数据的方法改为使用dataBase
        var fnBody = `return this.__dataBase.${methodName}()
                    .then(function(data){
                        return this.__handle_${propertyName}?_this.handle_${propertyName}(data):data;
                    })`;
        this[`get_${propertyName}`] = new Function(fnBody);
    }

    __handle_name(name) {
        //do somthing with name...
        return name;
    }

    __handle_age(age) {
        //do somthing with age...
        return age;
    }
}
var userDataBase = new UserDataBase();
var user = new User("123", userDataBase);
```

这样即使用户数据有一万种不同的属性字段，只要保证 `UserDataBase` 中良好地定义了对应的拉取方法，我们的 `User` 就能自动生成对应的方法。

这也就是元编程的优点之一，程序可以根据传入参数/对象的不同，动态地生成对应的程序，从而减少大量冗余的代码。

------
#进阶（五）

现在程序里还有点小瑕疵：

```js
//用户数据中不存在www字段，若这样执行会报错：
user.get_www(); //user.get_www is not a function
```

现在我们要保证像上面那样执行任意的 `user.get_xxxx()` ，程序不会报错，而是返回 `false`：

```js
//用户数据中不存在www字段：
user.get_www(); // => false
```

Javascript 里缺少了 Ruby 中 `method_missing` 这样黑科技的内核方法，但是我们可以通过 ES6 的 Proxy 特性来模拟：

```js
function createUser(id, userDataBase) {
    return new Proxy(new User(id, userDataBase), {
        get: (target, property) => (typeof(target[property]) === "function" ? target[property] : () => false)
    })
}

var userDataBase = new UserDataBase();
var user = createUser("123", userDataBase);

user.get_name() => // fetch name data
user.get_wwwwww() // => false
```
------
#总结

其实这里的 DEMO 只是元编程的一个小应用，下一篇文章里我们会通过元编程实现一个简单的表单验证 DSL ：

```js
//类似
form.name["is not empty"]["length is between",1,20] // => true or false
```

------

#参考

[来来来，咱么元编程入个门](http://zhuanlan.zhihu.com/p/20754002)

[元编程之javascript](http://www.cnblogs.com/liuyanlong/archive/2013/05/27/3102161.html)

[JavaScript 元编程之ES6 Proxy](https://www.phodal.com/blog/javascript-dsl-meta-programming-use-proxy/)