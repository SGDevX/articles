---
layout: post
title:  "Html5（SSE）实现方案"
date:   2016-08-09 21:00:11 +0800
categories: Http
tags:   Web Html Html5 Php Nodejs
toc: true
comments: true
---

先介绍一下Html5的新特性，在Html5中，定义了很多很有意思的特性，然而由于浏览器的瓶颈，很多特性并没有很快被支持，尤其是IE（元凶）。比如这个SSE(Server-Sent Event），在几年前很多浏览器还并不支持，随着时间的推移，技术的进步，才渐渐进入人们的视线中。

Html5中，一共推出了两种长连接方案，一个是SSE，一个是websocket，SSE是偏向简洁简单的实现，websocket是偏向复杂的交互的，两种实现方法中的负载情况我还没有评测过，后续可以用一些时间测试一下，看看效果如何。

下面就来详细介绍一下我的SSE学习过程。

# SSE基本原理 #

SSE说的其实就是服务器端推送特性，基本的实现就是用专门定义的Content-Type: text/event-stream 事件流，将服务器端的内容推送给web页，之所以叫服务器端推送，是因为他说的方法都是单向的，只能服务器端推送，web页只能被动的接收，而不是像websocket是双向互相通信的，不过SSE定义的一些方法其实是可以实现简单的双向通信的，后面我会专门提到。

------------------------图-----------------------------

# SSE基本实现 #


``` html
		<!doctype html>
		<html>
		    <head>
		        <meta charset="UTF-8">
		        <title>basic SSE 1</title>
		    </head>
		    <body>
		        <pre id = "res">init...</pre>
		    </body>
		    <script>
		        var es = new EventSource("sse.php");
		        es.addEventListener("message",function(e){
		            document.getElementById("res").innerHTML += "\n"+e.data;
		        },false);
		    </script>
		</html>
```

逻辑非常简单，声明一个EventSource指向一个服务器端接口即可，当服务器端返回内容的时候，onmessage接口会响应，处理对应的方法即可，例子中是直接贴出返回内容。


``` php
	<?php
		header('Content-Type: text/event-stream');
		header('Cache-Control: no-cache');
		echo "data: ". date("Y-m-d H:i:s") ."\n\n";
	?>
```

这里值得提出的是，返回必须是“data:”开头，而且必须是Content-Type: text/event-stream，最终结尾必须是两个\n，这样才会被识别为有效返回，可以理解为SSE的规则吧，在不同的浏览器上，可能会看到不同的结果，chrome是3s一返回，FF是5s一返回，很多文章也写过，这个时间是浏览器默认设置的，其实不然，SSE协议中还定义了repeat返回的时间间隔，这个也是可以设置的。


``` php
	<?php
	header('Content-Type: text/event-stream');
	header('Cache-Control: no-cache');
	echo "retry:3000\n";//毫秒
	echo "data: ". date("Y-m-d H:i:s") ."\n\n";
	?>
```

# SSE协议内容 #
分为以下几个部分：

1. data:（内容）  
2. retry:(重试时间)  
3. event:(add|remove|message)(推送事件类型，对应html捕捉事件)
4. httpcode 204（中断请求）
5. id:(event-id)  

1.2.之前已介绍过，就不说了。

3.event指的是你通知web端，你这次的返回是什么事件类型，此类型可自定义。
例如：event:hello\n,
js中只要监听hello事件即可获取相关返回。

4.httpcode 204 指的是，当你希望web端断开连接的时候，只需要返回204，web端就不会再向服务器端发送请求，表示两端的连接彻底断开。

5.id，这个我们详细说说

前面说到SSE是服务器端推送事件，基本可以理解为单向传输，但通过id这个参数，其实是可以传输一些内容到服务器端的，进行简单的交互。

下面是例子：

``` php
	<?php
	    header('Content-Type: text/event-stream');
		header('Cache-Control: no-cache');
		$time = date('H:i:s');
		$i = isset($_SERVER["HTTP_LAST_EVENT_ID"])?$_SERVER["HTTP_LAST_EVENT_ID"]:0;
		while(true){
			$i++;
			$time2 = date('H:i:s');
			$rand = rand(0,1000000);
			if($rand < 2){
				echo "id:".$i."\n";
				echo "data:hello:".$rand."-num:".$i."-date1:".$time."-date2:".$time2."\n\n";
				flush();
				break;
			}
			if($i > 10000000){
				@header('HTTP/1.1 204 No Content');
				break;
			}
		}
	?>
```

html代码还是保持不变即可，返回的时候我们带上id，下次请求web端就会带上HTTP_LAST_EVENT_ID这个http参数过来，通过php的$_SERVER["HTTP_LAST_EVENT_ID"]就能获取到相应的值，代码里就是从这个id开始往后计算，类似自增id遍历。

# SSE答疑解惑 #

有人可以会问了，这些例子都是请求直接断开，然后重试再次请求的，这样的例子和服务器端推送有什么关系呢？还不是类似频繁重试的逻辑，和老的http请求逻辑没什么太大区别，我刚开始阅读相关资料的时候也有类似的疑惑，但实际上并不是的。

只要你的服务器端可以在保持连接的状态下推送数据（data:111111\n\n,php可以使用ob_flush flush），就可以在保持连接的状态下把数据发送过去，并且web端可以正常接收并处理，不过这种情况php就不太擅长处理了，还是用nodejs类似来做会方便很多，而且nodejs在并发请求下效率更高，比较推荐。

# SSE应用场景 #

类似这种逻辑可以实现挺多需求的，例如弹幕，每条弹幕有自己的id，写入redis队列中，SSE获取redis中最新的弹幕信息返回给web端，并且带上最新的弹幕id，下次web端请求的时候将这个id带上来，服务器端按照上次的id返回redis池中最新的那些弹幕即可，简单高效，逻辑清晰。

更简单一些的需求场景是，二维码扫描支付，打开页面之后，web端和服务器端建立SSE，当用户扫描支付成功之后，服务器端收到消息通知web端，web端再显示支付成功等逻辑。

SSE的好处就是简单便捷，服务器端不需要额外配置，对服务器压力也比较小，缺点是现在IE还不支持（汗。。），web端无法把复杂的消息推送给服务器端，总体来讲，在一些小型的项目需求中，SSE是首选，优于websocket。

