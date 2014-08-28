---
layout: post
title: "使用Node.js开发RESTful WebService"
date: 2014-08-07 19:26:41 +0800
comments: true
categories: iOS
---
<!--more-->
##Node.js的Hello World
```javascript
var http = require('http');

//当有http请求发送过来时，就会调用此回调函数
http.createServer(function (req, res) {
    res.writeHead(200, {'Content-Type': 'text/plain'});
    res.end('Hello World\n');
}).listen(80, "127.0.0.1");
```
##Node.js是什么
在Chrome V8引擎上进行扩展从而生成的一个`Javascript运行环境/引擎`。一般JS都运行在客户端，Node.js使得JS可以在服务器端运行。
##Node.js的特点
`1.事件驱动、异步编程`  
在传统的网络编程中会用到回调函数，比如当socket资源达到某一状态时注册的回调函数就会执行。以Node.js中的Net模块为例，net.Socket对象就有connect、data、end等事件，开发者可以根据自己的业务逻辑注册响应的回调函数到这些事件。在代码中这些事件看上去是顺次注册的，但是它们并`不依赖自身出现的顺序`，而是在相应事件发生时触发。  
这两个特点得益于Node.js采用的语言是`Javascript`，Javascript的`匿名函数`和`闭包`特性非常适合事件驱动、异步编程。  
`2.高性能`  
选择C++和V8而不选择Ruby或其他虚拟机就是出于性能考虑。  
事实上Node.js是以`单进程、单线程`模式运行的，事件驱动机制是Node.js通过对内部单线程高效率的维护时间循环队列来实现，没有多线程的资源占用和上下文切换。面对大规模http请求，Node.js凭借事件驱动搞定一切。实验表明Node.js的事件驱动机制和单进程单线程模式导致CPU占用很高。现在已经有一些支持多进程的模块。  
##使用Express  
[Express](http://expressjs.com/)是基于Node.js的web开发框架，它被作为Node.js的一个module导入项目。下面借助Express来实现简单的REST API，这个主题的内容来自[这里](http://blog.modulus.io/nodejs-and-express-create-rest-api)。什么是`REST API`？可以看[维基](https://en.wikipedia.org/wiki/Representational_state_transfer#RESTful_web_APIs)。简单来说是由GET, PUT, POST和DELETE四种类型的方法组成。  
`1.集成Express到项目`  
修改`package.json`然后使用`npm install`来集成。
```javascript
{
    "name": "first-express-app",
    "description": "a fine piece of express art",
    "version": "0.0.1",
    "dependencies": {
        "express": "3.x"
    }
}
```
上例中会有警告，没有README之类的，添加private为true就没有警告了。  
`2.Express版的Hello World`  
```javascript
var express = require('express');
var app = express(); //app是Express server的一个实例

app.get('/', function(req, res) {
    res.type('text/plain');
    res.send('Hello World from Express');
});

//使用process.env.PORT就是说如果设置了PORT环境变量那就会用设置的端口，否则就是自己设定的3000
app.listen(process.env.PORT || 3000);

console.log('Server is running...');
```
然后`node filename.js`并在浏览器访问`http://localhost:3000`即可。  
`3.添加route实现新的API`  
route可以译作路由，就是访问网站时站点名后加的参数等等，用于实现在网站内跳转。在上面的代码中添加代码
```javascript
var quotes = [
{ author : 'Audrey Hepburn', text : "Nothing is impossible, the word itself says 'I'm possible'!"},
{ author : 'Walt Disney', text : "You may not realize it when it happens,  
            but a kick in the teeth may be the best thing in the world for you"},
{ author : 'Unknown', text : "Even the greatest was once a beginner. Don't be afraid to take that first step."},
{ author : 'Neale Donald Walsch', text : "You are afraid to die, and you're afraid to live. What a way to exist."}
];

//bodyParser中间件，可以parse请求的body，然后把body传给request的body property
//用于解析POST请求的内容
app.use(express.bodyParser());

app.get('/', function(req, res) {
    res.json(quotes);
});

app.get('/quote/random', function(req, res) {
    var id = Math.floor(Math.random() * quotes.length);
    var q = quotes[id];
    res.json(q);
});

//支持传入参数
//http://localhost:3000/quote/2
app.get('/quote/:id', function(req, res) {
    if(quotes.length <= req.params.id || req.params.id < 0) {
        res.statusCode = 404;
        return res.send('Error 404: No quote found');
    }

    var q = quotes[req.params.id];
    res.json(q);
});

//curl -v -H "Accept: application/json" -H "Content-type: application/json" -X  
//POST -d '{"author": "daryl", "text": "daryls post"}' http://localhost:3000/quote/
app.post('/quote', function(req, res) {
    if(!req.body.hasOwnProperty('author') || !req.body.hasOwnProperty('text')) {
        res.statusCode = 400;
        return res.send('Error 400: Post syntax incorrect.');
    }

    var newQuote = {
        author : req.body.author,
        text : req.body.text
    }; 

    quotes.push(newQuote);
    res.json(true);
});

//curl -I -X DELETE http://localhost:3000/quote/7
app.delete('/quote/:id', function(req, res) {
    if(quotes.length <= req.params.id) {
        res.statusCode = 404;
        return res.send('Error 404: No quote found');
    }

    quotes.splice(req.params.id, 1);
        res.json(true);
});
```





##参考文献
1.[深入浅出Node.js（一）：什么是Node.js](http://www.infoq.com/cn/articles/what-is-nodejs)  系列文章在[这里](http://www.infoq.com/cn/master-nodejs)  
2.[NODE.JS AND EXPRESS - CREATING A REST API](http://blog.modulus.io/nodejs-and-express-create-rest-api)  

