---
layout: post
title:  "Html5（WebSocket）实现方案"
date:   2016-10-12 23:12:00 +0800
categories: Http
tags:   Web Html Html5 Php Nodejs WebSocket
toc: true
comments: true
---

长久以来网络通信的协议都没有很好的解释长连接，阻碍了时代的发展和进步，html5 WebSocket的出现可以说是解决了这一老大难问题。所以WebSocket既是划时代的产物，也是时代进步必然的产物。html5定义了WebSocket，而随后java等大牛产品也实现了WebSocket协议，可以说这是对WebSocket协议的认可。WebSocket实现了浏览器与服务器端的全双工通信(full-duplex)，一开始的握手需要借助HTTP完成。


# WebSocket基本原理 #
WebSocket是一个协议，一个基于http协议，但是却截然不同的存在。WebSocket是持久化连接，但在第一步建立持久化握手的时候是通过http协议实现的。

第一步：
发送一条http请求，附带如下信息
Upgrade: websocket
Connection: Upgrade
告知服务器端，我要进行WebSocket通信，请许可，并转换为WebSocket协议。

第二步：
服务器端返回许可，告知客户端可以进行WebSocket通信。

第三步：
客户端再度发起请求，和服务器端建立WebSocket通信服务。

之后，客户端会每隔几秒进行一次心跳请求，用来确认服务器端还存活，如果发现服务器端断开了连接，则客户端也会停止WebSocket，服务终止。

# WebSocket基本实现 #
对WebSocket支持最好的，最广泛的就是近几年出现的Nodejs了，而Nodejs中对WebSocket支持最好的扩展就是socket.io了，我们这一章节的例子都是以nodejs/socket.io 作为基础的。

``` html
	<html>
	<head>
	    <title></title>
	    <script src="http://iploc:8111/socket.io/socket.io.js"></script>
	    <script type="text/javascript">
	        function doit() {
	            var socket = io.connect('http://iploc:8111');//请求服务器端，建立webSocket连接。
	            socket.on('new event', function (data) {//监听服务器端的'new event'事件。
	                console.log(data.hello);
	                socket.emit('other event', { hello:'other world' });//向服务器端发送'other event'事件。
	            });
	        }
	    </script>
	</head>
	<body>
	<button id='btn' onclick="doit()">click me</button>
	</body>
	</html>
```



``` nodejs
	var app = require('express')()
		, server = require('http').createServer(app)
	    , io = require('socket.io').listen(server);
	server.listen(8111);
	io.sockets.on('connection', function (socket) {//监听客户端连接
		socket.emit('new event', { hello: 'world' });//当客户端连接的时候发送事件new event给客户端。
		socket.on('other event', function (data) {//监听客户端事件other event
			console.log(data.hello);
		});
	});
```

这个例子只是最简单的双工通信的示例，也是Websocket最重要的基本功能，在html5推出之前大家梦寐以求的长连接通信方案，简单几行代码就搞定了，不得不感慨世界技术的进步。


# WebSocket分组管理 #
WebSocket的每个连接其实都有一个唯一独立id，利用这个id可以将不同的用户分组，实现“频道”的概念，这也是被普遍使用的功能之一。


``` html
	<html>
	<head>
	    <title></title>
	    <script src="http://iploc:8111/socket.io/socket.io.js"></script>
	    <script src="jquery.js"></script>
	    <script type="text/javascript">
	            var socket = io.connect('http://iploc:8111');
	            socket.on('receiveRedisRoom', function (data) {
	                console.log(data);
					$("#output").append(data+"\n");
	            });
	    </script>
	</head>
	<body>
	<div id="output">output-content<br/>
	</div>
	</body>
	</html>
```

``` nodejs
	var app = require('express')()
		, server = require('http').createServer(app)
    	, io = require('socket.io').listen(server);
	server.listen(8111);
    io.on("connection", function(socket){
		socket.join('room 1');
		socket.emit('receiveRedisRoom','join room 1');  
    });
```
实际上，每个客户端请求服务器建立长连接的时候，都会被赋予一个独立的组，组名就是用他的id来标识的，如果需要锁定特定的客户端，通过id来找到组就可以操作了。

``` nodejs
	var app = require('express')()
	, server = require('http').createServer(app)
    , io = require('socket.io').listen(server);
	server.listen(8111);
    io.on("connection", function(socket){
		//以下三行可以达到同样的效果
		socket.emit('receiveRedisRoom',msg);  
		io.sockets.sockets[socket.id].emit('receiveRedisRoom',msg);
		io.sockets.adapter.rooms[socket.id].emit('receiveRedisRoom',msg);
    });
```


# WebSocket+redis通信 #
假设我们做的是一个聊天室，这个时候用户都连接了服务器进行聊天，那后台是什么呢？具体怎么操作呢？这个还是要借助别的功能来实现，redis就是一种很方便的工具。使用Nodejs关注redis的publish，然后后台admin向redis发送信息就可以使得信息发送给指定用户了，很方便，流程也很顺畅。


```html
	<html>
	<head>
	    <title></title>
	    <script src="http://iploc:8111/socket.io/socket.io.js"></script>
	    <script src="jquery.js"></script>
	    <script type="text/javascript">
	            var socket = io.connect('http://iploc:8111');
	            socket.on('receiveRedisMsg', function (data) {
	                console.log(data);
					$("#output").append(data+"\n");
	            });
	    </script>
	</head>
	<body>
	<div id="output">output-content<br/>
	</div>
	</body>
	</html>
````

``` nodejs
	var app = require('express')()
	, server = require('http').createServer(app)
    , io = require('socket.io').listen(server);
	server.listen(8111);
    var redis = require("redis");
    var sub = redis.createClient("6379","redis.ip");
    sub.subscribe("chat"); // 订阅chat频道
    sub.subscribe("chat2"); // 订阅chat频道
    console.log("redis example is running!");            
    io.on("connection", function(socket){
        sub.on("message", function(channel, msg){ // chat频道一旦接收到消息msg,则立即向socket.io连接中发送该msg数据.
            console.log("redis on message", msg,channel,socket.id,io.sockets.adapter.rooms);
            socket.emit("receiveRedisMsg", msg);
        })
    })
```

其实应用redis已经超出了WebSocket的范畴，而是nodejs/socket.io实现的功能了。WebSocket本身只是一个协议，之后的种种衍生物都是为了让WebSocket更好的便于使用而产生的，也从侧面证明了WebSocket的意义。


# WebSocket 应用场景 #

WebSocket应用场景非常之多，从轻量级的H5，聊天室，oa，到复杂的游戏，都有它的身影。总的来说，需要长连接，实时交互逻辑的都可以考虑使用WebSocket。我往往喜欢把sse和WebSocket拿来对比，这两种协议都是html5新推出的，又都是应用于长连接领域的，很相似。

比如，一个用户在PC上淘宝下单，用手机扫码支付，手机支付成功，则PC上页面也自动跳转为支付成功页。你们觉得这个逻辑应该用以上哪种协议呢？

这个场景是用到长连接的，PC页面需要一直请求服务器端，等待用户支付成功，服务器端将支付成功的消息推送到PC页面，PC页面才能跳转到支付成功页。我个人觉得这个场景，用sse更好，因为整个逻辑中PC页面是不需要通知服务器端的，而只需要服务器端通知PC页面，所以sse作为单向通信，应该是更加合适的，也能更加节省服务器端资源。之后我会再做一下两种协议的资源消耗对比，看我的推论是否正确。