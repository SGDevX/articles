---
layout: post
title:  "旧版本html支持的长连接方案"
date:   2016-08-17 21:00:05 +0800
categories: Http
tags:   Web Html Html5 Php Nodejs
toc: true
comments: true
---

# 旧版本html长连接实现方案简介 #
首先，我们要搞懂长连接到底要做什么，需求是什么?所谓长连接肯定是为了做一些短连接满足不了的场景，在客户端和服务器端中间搭建一条即时通讯的桥梁。所以，实际上只要能满足即时通讯，基本上都是符合要求的，这里长连接实现方案其实还细分成几种。

1. 短轮询：每次请求都是短连接，客户端会以一个比较高的频率频繁请求服务器，服务器有新信息就会即时返回。这种方案会占用比较大的服务器资源，并且返回实际上并不是真的足够及时，因为会根据请求频率是有一定间隔的。
2. 长轮询：每个请求是长连接，请求到服务器之后，直到服务器有返回才会断开连接并把数据返回给客户端，再按照一定的时间间隔，继续建立长连接。这种方案比短轮询占用更大的服务器资源，但带宽占用会小（因为省略了很多http头信息传递），并且返回是相对及时的（因为服务器有新内容会立刻返回），也有一些小瑕疵，因为从请求断开到重连，中间还是会有时间间隔的，这段时间出现的新内容，并没有足够快的返回给客户端。
3. 长连接：请求是一直保持连接的，服务器有内容，就直接push给客户端，但连接并不断开。这样的服务器资源消耗也是很大，但能够保证理论上的足够即时。


# 短轮询 #

接下来，我们再来详细看一下代码实现，依旧是html+php，低版本没有浏览器兼容性问题。实现方案主要分成ajax，js添加，iframe方式。

## ajax方式 ##


web端：

``` html
	<html>
	<head>
	<title>html lc test 1</title>
	<script src="jquery.js"></script>
	</head>
	<body>
	<div id="res"></div>
	</body>
	<script type="text/javascript">
		setInterval("longCnt()",1000);
		function longCnt(){
			var obj= $.ajax({url:"e1.php",async:false});
			  $("#res").append(obj.responseText);
		}
	</script>
	</html>
```

服务器端：

``` php
	<?php
	echo "hello:".date("Y-m-d H:i:s")."<br/>\n";
	?>
```

代码挺简单的，直接用ajax方式请求即可，这种方法不能跨域。同时jquery的ajax的jsonp方式，是可以跨域的。

## js添加方式 ##

web端：

``` html
	<html>
	<head>
	<title>lc html test 2</title>
	<script src="jquery.js"></script>
	</head>
	<body>
	<div id="res"></div>
	</body>
	<script type="text/javascript">
		setInterval("longCnt()",1000);
		function longCnt(){
			var js = document.createElement('script');
			js.language = 'javascript';
			js.setAttribute('src', 'e2.php');
			document.body.appendChild(js);
		}
	</script>
	</html>
``` 

服务器端：

``` php
	<?php
	echo '$("#res").append("hello:'.date("Y-m-d H:i:s").'<br/>");';
	?>
```

利用添加script标签来引用新的内容，内容必须是在html中定义过的方法，更完美的做法是给script标签添加一个id，在每次请求新的script的时候，把旧标签删除，减少页面压力，不然运行时间长了，页面过于臃肿。而且，这种方法可以跨域，也有不少黑客的代码是用类似的方法来实现的。

## iframe方式 ##

web端：

``` html
	<html>
	<head>
	<title>lc html test 3</title>
	<script src="jquery.js"></script>
	</head>
	<body>
	<div id="res"></div>
	</body>
	<script type="text/javascript">
		setInterval("longCnt()",1000);
		function longCnt(){
			var iframe = document.createElement('iframe');
			iframe.setAttribute('src', 'e3.php');
			iframe.style.display = "none";
			document.body.appendChild(iframe);
		}
	</script>
	</html>
```

服务器端：

``` php
	<?php
	echo "<script>";
	echo 'window.parent.document.getElementById("res").innerHTML += "hello:'.date("Y-m-d H:i:s").'<br/>";';
	echo "</script>";
	?>
```


这种方案应该是最原始的了，在ajax还没有出现之前，这种方案应该是最流行的。同样，最好在请求新的iframe的时候把旧的删除掉，减轻页面压力，这种方案也是不能跨域的。


#  长轮询  #

长轮询的代码其实和上面基本一致，只是在服务器端层面，不要第一时间返回数据即可。

服务器端：

``` php
	<?php
	while(true){
		if(haveNewMessage){
			echo "message to client;";
			break;		
		}
	}
	?>
```


#  长连接  #

长连接php不是很擅长了，很多年前的php+apache的时候，可以用ob_flush();flush();来实现实时输出。但nginx有了fcgi的输出缓冲区（在服务器返回给客户端之前，先把内容放入缓冲区，等数据全部发送完毕，或者到了一定临界值的时候才整体打包发送给客户端，这样的好处是可以减少网络请求），除非把cgi缓冲区设置的特别小，不然phpcgi+nginx是无法做到实时返回数据的，所以长连接一般都用其他的语言来代替，例如nodejs。



# 总结 #

旧html的方式是在旧的基础架构上不得已而为之的产物，对网络开销，服务器损耗都比较大，随着技术的进步，新规则必定会代替旧的规则。后续我会介绍html5在长连接方向上的努力，SSE和websocket都是很好的新规则，后面的章节会一一介绍。