## ![koa](http://ww4.sinaimg.cn/large/61ff0de3gw1ebtqstxfuvj202g01awea.jpg) 中文文档 ![koa@npm](https://badge.fury.io/js/koa.png) [![Build Status](https://travis-ci.org/koajs/koa.png)](https://travis-ci.org/koajs/koa)

Koa，下一代 Node.js web 框架

### koa 简介

由 Express 原班人马打造的 koa，致力于成为一个更小、更健壮、更富有表现力的 Web 框架。通过组合不同的 generator，使用 koa 编写 web 应用，可以免除重复且繁琐的回调，并极大地提升常用错误处理的效率。Koa 不会绑定任何中间件在内核方法中，它仅仅提供了一个轻量优雅的函数库，使得编写 Web 应用变得得心应手。

### 安装 koa

koa 依赖支持 generator 的 Node 环境，准确来说，是 `node >= 0.11.9` 的环境。

````
$ npm install koa
````

安装完成后，确保使用 `$ node app.js --harmony` 即，harmony 模式运行程序。

为了方便，你可以将 node 设置为默认启动 harmony 模式的别名：

````
alias node='node --harmony'
````

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

Koa 中间件以一种非常传统的方式级联起来，所以你可能会非常熟悉这种写法。在以往的 Node 开发中，频繁使用回调不太便于表达代码逻辑的表现力，而在 Koa 中，我们可以写出真正具有表现力的中间件。与 Connect 实现中间件的方法想对比，Koa 的做法不会简单的将控制权移交给一个又一个的中间件直到程序返回，Koa 执行代码的方式有点像回形针，用户请求通过中间件时，遇到 `yield next` 关键词时，会被传递到下一个符合请求的路由（downstream），在 `yield next` 捕获不到下一个中间件时，逆序返回继续执行代码（upstream）。

下边这个例子展现了使用这一特殊方法书写的 Hello World 范例：一开始，用户的请求通过 x-response-time 中间件和 logging 中间件，这两个中间件记录了一些请求细节，然后「穿过」 response 中间件一次，最终结束请求，返回 「Hello World」。

当程序运行到 `yield next` 时，代码流会暂停执行这个中间件的剩余代码，转而切换到下一个被定义的中间件执行代码，这样切换控制权的方式，被称为 
downstream，当没有下一个中间件执行 downstream 的时候，代码将会逆序执行。

````javascript
var koa = require('koa');
var app = koa();

// x-response-time
app.use(function *(next){
  // (1) 进入路由
  var start = new Date;
  yield next;
  // (5) 再次进入 x-response-time 中间件，记录2次通过此中间件「穿越」的时间
  var ms = new Date - start;
  this.set('X-Response-Time', ms + 'ms');
  // (6) 返回 this.body
});

// logger
app.use(function *(next){
  // (2) 进入 logger 中间件
  var start = new Date;
  yield next;
  // (4) 再次进入 logger 中间件，记录2次通过此中间件「穿越」的时间
  var ms = new Date - start;
  console.log('%s %s - %s', this.method, this.url, ms);
});

// response
app.use(function *(){
  // (3) 进入 response 中间件，没有捕获到下一个符合条件的中间件，传递到 upstream
  this.body = 'Hello World';
});

app.listen(3000);
````
在上方的范例代码中，中间件以此被执行的顺序已经在注释中标记出来。你也可以自己尝试运行一下这个范例，并打印记录下各个环节的输出与耗时。

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

除非应用执行环境被配置为 `"test"`，Koa 都将会将所有错误信息输出到 stderr，和 Connect 一样，你可以自己定义一个「错误事件」来监听 Koa app 中发生的错误：

````javascript
app.on('error', function(err){
  log.error('server error', err);
});
````

当任何 req 或者 res 中出现的错误无法被回应到客户端时，Koa 会在第二个参数传入这个错误的上下文：

````javascript
app.on('error', function(err, ctx){
  log.error('server error', err, ctx);
});
````

如果任何错误有可能被回应到客户端，比如当没有新数据写入 socket 时，Koa 会默认返回一个 500 错误，并抛出一个 app 级别的错误到日志处理中间件中。

### 应用的上下文（Context）

Koa 的上下文封装了 request 与 response 对象至一个对象中，并提供了一些帮助开发者编写业务逻辑的方法。为了方便，你可以在 `ctx.request` 和 `ctx.response` 中访问到这些方法。

每一个请求都会创建一段上下文。在控制业务逻辑的中间件中，上下文被寄存在 `this` 对象中，你可以这样访问：

````javascript
app.use(function *(){
  this; // 上下文对象
  this.request; // Request 对象
  this.response; // Response 对象
});
````

#### Request 对象

ctx.request 对象包括以下属性和别名方法，详见 [Request](#request) 章节

- ctx.header
- ctx.method
- ctx.method=
- ctx.url
- ctx.url=
- ctx.path
- ctx.path=
- ctx.query
- ctx.query=
- ctx.querystring
- ctx.querystring=
- ctx.length
- ctx.host
- ctx.fresh
- ctx.stale
- ctx.socket
- ctx.protocol
- ctx.secure
- ctx.ip
- ctx.ips
- ctx.subdomains
- ctx.is()
- ctx.accepts()
- ctx.acceptsEncodings()
- ctx.acceptsCharsets()
- ctx.acceptsLanguages()
- ctx.get()

#### Response 对象

ctx.response 对象包括以下属性和别名方法，详见 [Response](#response) 章节

- ctx.body
- ctx.body=
- ctx.status
- ctx.status=
- ctx.length=
- ctx.type
- ctx.type=
- ctx.headerSent
- ctx.redirect()
- ctx.attachment()
- ctx.set()
- ctx.remove()
- ctx.lastModified=
- ctx.etag=

#### 上下文对象中的其他 API

- ctx.req: Node.js 中的 request 对象
- ctx.res: Node.js 中的 response 对象
- ctx.app: app 实例
- ctx.cookies.get(name, [options]) 对于给定的 name ，返回响应的 cookie
- ctx.cookies.set(name, value, [options]) 对于给定的参数，设置一个新 cookie
    - options
        * `signed`
        * `expires`
        * `path`
        * `domain`
        * `secure`
        * `httpOnly`
- ctx.throw(msg, [status]) 抛出常规错误的辅助方法，以下几种写法都有效：

````javascript
this.throw(403)
this.throw('name required', 400)
this.throw(400, 'name required')
this.throw('something exploded')
````

实际上，ctx.throw 是这些代码片段的简写方法：

````javascript
var err = new Error('name required');
err.status = 400;
throw err;
````

需要注意的是，ctx.throw 创建的错误，均为用户级别错误（标记为err.expose），会被返回到客户端。

### Request

`ctx.request` 对象是对 Node 原生请求对象的抽象包装，提供了一些非常有用的方法。详细的 Request 对象 API 如下：

#### req.header

#### req.method

#### req.method=

设置 req.method ，用于实现输入 methodOverride() 的中间件

#### req.length

返回 req 对象的 `Content-Length` (Number)

#### req.url

#### req.url=

#### req.path

#### req.path=

#### req.querystring

返回类似 Express 中 req.query 查询字符串，去除了头部的 `'?'`

#### req.querystring=

#### req.search

返回类似 Express 中 req.query 查询字符串，包含了头部的 `'?'`

#### req.search=

#### req.host

#### req.type

返回 req 对象的 `Content-Type`

#### req.query

返回经过解析的查询字符串，类似 Express 中的 req.query

#### req.query=

#### req.fresh

检查 req 的缓存是否是最新。当缓存为最新时，可编写业务逻辑直接返回 `304`

#### req.stale

和 req.fresh 返回的结果正好相反

#### req.protocol

#### req.secure

#### req.ip

#### req.ips

#### req.subdomains

返回 req 对象中的子域名数组。

#### req.is(type)

判断 req 对象中 Content-Type 是否为给定 type 的快捷方法：

````javascript
// With Content-Type: text/html; charset=utf-8
this.is('html'); // => 'html'
this.is('text/html'); // => 'text/html'
this.is('text/*', 'text/html'); // => 'text/html'

// When Content-Type is application/json
this.is('json', 'urlencoded'); // => 'json'
this.is('application/json'); // => 'application/json'
this.is('html', 'application/*'); // => 'application/json'

this.is('html'); // => false
````

#### req.accepts(type)

判断 req 对象中 Accept 是否为给定 type 的快捷方法：

````javascript
// Accept: text/html
this.accepts('html');
// => "html"

// Accept: text/*, application/json
this.accepts('html');
// => "html"
this.accepts('text/html');
// => "text/html"
this.accepts('json', 'text');
// => "json"
this.accepts('application/json');
// => "application/json"

// Accept: text/*, application/json
this.accepts('image/png');
this.accepts('png');
// => undefined

// Accept: text/*;q=.5, application/json
this.accepts(['html', 'json']);
this.accepts('html', 'json');
// => "json"
````

#### req.acceptsEncodings(encodings)

判断客户端是否接受给定的编码方式的快捷方法，当有传入参数时，返回最应当返回的一种编码方式

````javascript
// Accept-Encoding: gzip
this.acceptsEncodings('gzip', 'deflate');
// => "gzip"

this.acceptsEncodings(['gzip', 'deflate']);
// => "gzip"
````

当没有传入参数时，返回客户端的请求数组：

````javascript
// Accept-Encoding: gzip, deflate
this.acceptsEncodings();
// => ["gzip", "deflate"]
````

#### req.acceptsCharsets(charsets)

使用方法同 req.acceptsEncodings(encodings)

#### req.acceptsLanguages(langs)

使用方法同 req.acceptsEncodings(encodings)

### Response

详细的 Response 对象 API

### 性能（Benchmarks）

挂载不同数量的中间件，wrk 得出 benchmarks 如下：

````
1 middleware
8367.03

5 middleware
8074.10

10 middleware
7526.55

15 middleware
7399.92

20 middleware
7055.33

30 middleware
6460.17

50 middleware
5671.98

100 middleware
4349.37
````
一般来说，我们通常要使用约50个中间件，按这个标准计算，单应用可支持 340,260 请求/分钟，即 20,415,600 请求/小时，也就是约 4.4 亿 请求/天。

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