## koa-guide

## koa 中文文档

koa: 下一代 Node.js web 框架

### koa 简介

由 Express 原班人马打造的 koa，致力于成为一个更小、更健壮、更富有表现力的 Web 框架。通过组合不同的 generator，使用 koa 编写 web 应用，可以免除重复且繁琐的回调，并极大地提升常用错误处理的效率。Koa 不会绑定任何中间件在内核方法中，它仅仅提供了一个轻量优雅的函数库，使得编写 Web 应用变得得心应手。

### 应用（Application）

一个 Koa Application（以下简称 app）由一系列 generator 中间件组成。按照编码顺序在栈内依次执行，从这个角度来看，Koa app 和其他中间件系统（比如 Ruby Rack 或者 Connect/Express ）没有什么太大差别，不过，从另一个层面来看，Koa 提供了一种基于底层中间件编写「语法糖」的设计思路，这让设计中间件变得更简单有趣。

在这些中间件中，有负责内容协商（content-negotation），缓存控制（cache freshness），反向代理（proxy support）和重定向等等功能常用中间件，但如前所述， Koa 内核并不会打包这些中间件，让我们先来看看 Koa 极其简单的 Hello World 程序吧：

````javascript
var koa = require('koa');
var app = koa();

app.use(function *(){
  this.body = 'Hello World';
});

app.listen(3000);
````

### 编写级联代码（Cascading）

Koa 中间件以一种非常传统的方式级联起来，所以你可能会非常熟悉这种写法。在以往的 Node 开发中，频繁使用回调不太便于表达代码逻辑的表现力，而在 Koa 中，我们可以写出真正具有表现力的中间件。与 Connect 实现中间件的方法想对比，Koa 的做法不会简单的将控制权移交给一个又一个的中间件直到程序返回，Koa 执行 downstream 协程，然后返回到 upstream 继续执行代码（?）。

下边这个例子展现了使用这一特殊方法书写的 Hello World 范例：一开始，用户的请求通过 x-response-time 中间件和 logging 中间件，这两个中间件记录了一些请求细节，然后「穿过」 response 中间件数次，最终，再运行到 response，结束请求，返回 「Hello World」。

当程序运行到 `yield next` 时，代码流会暂停执行这个中间件的剩余代码，转而切换到下一个被定义的中间件执行代码，这样切换流的方式，被称为 
downstream，当没有下一个中间件执行 downstream 的时候，代码会恢复正常的方式被执行。

````javascript
var koa = require('koa');
var app = koa();

// x-response-time

app.use(function *(next){
  var start = new Date;
  yield next;
  var ms = new Date - start;
  this.set('X-Response-Time', ms + 'ms');
});

// logger

app.use(function *(next){
  var start = new Date;
  yield next;
  var ms = new Date - start;
  console.log('%s %s - %s', this.method, this.url, ms);
});

// response

app.use(function *(){
  this.body = 'Hello World';
});

app.listen(3000);
````

### 应用配置（Settings）

应用的配置是 app 实例的属性。目前来说，Koa 的配置项如下：

- app.name 应用名称
- app.env 执行环境，默认是 `NODE_ENV` 或者 `"development"` 字符串
- app.proxy 决定了什么 `proxy header` 参数会被加到信任列表中
- app.subdomainOffset 被忽略的 `.subdomains` 列表(?) [2]
- app.jsonSpaces 默认的 JSON 响应空间(?) [2]
- app.outputErrors 是否输出错误堆栈（`err.stack`）到 `stderr` [当执行环境是 `"test"` 的时候为 `false`]

### 常用方法

#### app.listen(...)

用于启动一个服务的快捷方法，以下范例代码在 3000 端口启动了一个空服务：

````javascript
var koa = require('koa');
var app = koa();
app.listen(3000);
````
app.listen 是 http.createServer 的简单包装，它实际上这样运行：

````javascript
http.createServer(app.callback()).listen(3000);
````

如果有需要，你可以在多个端口上启动一个 app，比如同时支持 HTTP 和 HTTPS：

````javascript
http.createServer(app.callback()).listen(3000);
http.createServer(app.callback()).listen(3001);
````

#### app.callback()

返回一个可被 `http.createServer()` 接受的程序实例，也可以将这个返回函数挂载在一个 Connect/Express 应用中。

#### app.use(function)

将给定的 function 当做中间件加载到应用中，详见 [中间件](#middleware) 章节

#### app.keys=

设置一个签名 Cookie 的密钥。这些参数会被传递给 [KeyGrip](https://github.com/jed/keygrip) 如果你想自己生成一个实例，也可以这样：

````javascript
app.keys = ['im a newer secret', 'i like turtle'];
app.keys = new KeyGrip(['im a newer secret', 'i like turtle'], 'sha256');
````
注意，签名密钥只在配置项 `signed` 参数为真是才会生效：

````javascript
this.cookies.set('name', 'tobi', { signed: true });
````

### 错误处理（Error Handling）

### 应用的上下文（Context）

### Contributing
- Fork this repo
- Clone your repo
- Install dependencies
- Checkout a feature branch
- Feel free to add your features
- Make sure your features are fully tested
- Open a pull request, and enjoy <3

### MIT license
Copyright (c) 2013 turing &lt;o.u.turing@gmail.com&gt;

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
THE SOFTWARE.

---
![docor](https://cdn1.iconfinder.com/data/icons/windows8_icons_iconpharm/26/doctor.png)
generated using [docor](https://github.com/turingou/docor.git) @ 0.1.0. brought to you by [turingou](https://github.com/turingou)