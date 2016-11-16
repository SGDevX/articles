---
layout: post
title:  "客户端旧版本长连接（nginx-push-stream-module）实现方案  "
date:   2016-11-09 20:32:00 +0800
categories: Http
tags:   Web Html Nginx
toc: true
comments: true
---

其实在html5出现之前，nginx已经给出了长连接的支持，只是性能上由于协议大环境的原因，可能不是很好。但肯定是比你自己用开发语言实现要好的多的，毕竟你开发语言实现的时候也是通过nginx转发的，直接使用nginx module相当于少了一层逻辑。
nginx-push-stream 是通过pub/sub来实现的，类似WebSocket里介绍到的redis订阅。后续工作组又在nginx-push-stream的基础上，开发了一套框架：nginx-push-stream-module，可以支持longpolling，eventsource，stream，后续与时俱进的支持了websocket协议。

# nginx-push-stream-module 安装 #
以下摘自官网 [官网链接](https://www.nginx.com/resources/wiki/modules/push_stream/)

``` shell
	# clone the project
	git clone http://github.com/wandenberg/nginx-push-stream-module.git
	NGINX_PUSH_STREAM_MODULE_PATH=$PWD/nginx-push-stream-module
	cd nginx-push-stream-module
	# build with 1.0.x, 0.9.x, 0.8.x series
	./build.sh master 1.0.5
	cd build/nginx-1.0.5
	# install and finish
	sudo make install
	# check
	sudo /usr/local/nginx/sbin/nginx -v
	    nginx version: nginx/1.0.5
	# test configuration
	sudo /usr/local/nginx/sbin/nginx -c $NGINX_PUSH_STREAM_MODULE_PATH/misc/nginx.conf -t
	    the configuration file $NGINX_PUSH_STREAM_MODULE_PATH/misc/nginx.conf syntax is ok
	    configuration file $NGINX_PUSH_STREAM_MODULE_PATH/misc/nginx.conf test is successful
	# run
	sudo /usr/local/nginx/sbin/nginx -c $NGINX_PUSH_STREAM_MODULE_PATH/misc/nginx.conf
```

# nginx-push-stream-module 的基本配置 #
nginx-push-stream-module是通过pub/sub实现的，长连接就是相当于你订阅(sub)了某个通道，服务器通过推送(pub)信息到这个通道，就相当于订阅了这个通道的所有人都能收到这条信息。

``` shell
location ~ /sub/(.*) {
    # activate subscriber (streaming) mode for this location
    push_stream_subscriber;
    # positional channel path
    set $push_stream_channels_path              $1;
}
```
这个是订阅的配置，请求这个url类似http://site/sub/1，如果不存在这个频道，则会创建频道1并订阅，否则就直接订阅频道1。

``` shell
location /pub {
    # activate publisher (admin) mode for this location
    push_stream_publisher admin;
    # query string based channel id
    set $push_stream_channel_id             $arg_id;
}
```
这个是推送的配置，发送内容到http://site/pub/1,所有订阅了频道1的用户都会收到消息。
比较死板的是只有这一种方式，不过满足简单需求是没有问题的。 

``` shell
location /channels-stats {
    # activate channels statistics mode for this location
    push_stream_channels_statistics;

    # query string based channel id
    set $push_stream_channel_id             $arg_id;
}
```
查看推送频道状态，能够获取一些频道的基本信息，还算比较好用。

``` shell
curl -s -v -X DELETE 'http://localhost/pub/1'
```
删除对应的频道，终止订阅。

# nginx-push-stream-module 的应用场景 #
这章介绍的nginx-push-stream-module，实际上和前几章介绍的长连接方案不太一样，前几章介绍的是协议，是实现长连接所定义的规则，而这章介绍的是应用这些协议所产生的技术方案。直接应用nginx有很多好处，最大的就是效率的提升，比用语言实现会好很多，缺点也说了，还是不够灵活，只能实现最基本的消息通信。

nginx保持了他一贯的简洁风格，只支持最重要的功能，始终把自己定位为高性能的HTTP和反向代理服务器，在实现众多功能和支持的同时，不忘初心，保持着良好的性能。

只要支持nginx的服务器都可以使用这个模块来实现长连接，这个有些废话了，与其着眼于优点，不如去想一下缺点，如果你的服务需要复杂的交互，只是简单的pub/sub不能表达你的意思的，那这种方式就不太适合你了，可能对有些复杂的手游网游就不太适合了。不过总的来说，绝大多数长连接服务更多的要求是实时性，至于复杂交互更多还是用请求专门的接口来完成的，所以nginx-push-stream-module的应用场景很广泛。