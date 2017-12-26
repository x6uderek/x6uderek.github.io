---
layout: page
title: 使用nodejs+socket.io与页面通讯
date: 2016-02-20 16:48:00
categories: blog
tags: [NodeJS,socket,"Long connection","web message pushing"]
description: 使用nodejs+socket.io与页面通讯
---


socket通信允许服务器主动推送消息给浏览器，这在一些场景（如聊天，展示异步任务的进度）很有用。下面总结一些使用方法。

**注意：以下文字基于nodejs v0.10.24，socket.io v0.9 ,其他版本以官方文档为准**

首先安装nodejs
1. Windows下直接下载安装包安装（<http://nodejs.org/download/>）
2. linux下可以自己编译也可以用包管理器（参考<https://github.com/joyent/node/wiki/Installing-Node.js-via-package-manager>）

然后安装socket.io，使用命令
```
npm install socket.io
```
Windows下安装包为我们提供了npm环境，如下图

![图1]({{ site.static }}img/049e01cb3d5ff79ec5d14bab9b6abacd.jpg)

socket.io会被安装在当前目录，如果要在项目里面使用，必须先 cd 到项目目录，不同的项目需要安装多次

这样会很怪，解决方法是把 `node_modules` 这个文件夹地址加到环境变量 `NODE_PATH` 中(没有就新建)

现在可以写段脚步测试一下socket.io有没有安装成功
```javascript
var io = require('socket.io');
```
用node运行一下，看看是否有错误

下面建立一个服务端脚本
```javascript
var http = require('http');  
var redis = require('redis');  
//加载配置文件，路径应该使用"/"而不是"\"(无论windows或者linux)  
var config = require('./config').Config.getConfig();  
//加载工具类  
var util = require('../common/util').Util;  
var server = http.createServer(function (request, response) {});  
var io = require('socket.io').listen(server);  
   
server.listen(config.server_port);  
//运行的来源地址  
io.set('origins','*:*');  
//传输方式  
io.set('transports',['websocket'/*,'flashsocket','htmlfile','xhr-polling','jsonp-polling'*/]);  
//用一个数组保存当前所有连接  
var socket_array = new Array();  
//namespace必须以"/"开头  
io.of(config.namespace).on('connection',function(socket){  
    //保存为全局变量  
    socket_array.push(socket);  
    console.log('client connected');  
    socket.on('message',function(event){  
        console.log('Received message from client!',event);  
    });  
    socket.on('disconnect',function(){  
        //当连接断开，将他从数组中删除  
        var search_index = util.array_search(socket,socket_array);  
        if(search_index != -1){  
            socket_array.splice(search_index,1);  
        }  
        console.log('client has disconnected',socket_array.length,'left');  
    });  
});  
server.on("close",function(){  
//服务停止事件  
});
```

上面的代码引用config.js配置文件
```javascript
function config(){  
    this.getConfig = function(){  
        return {  
            server_port:8080,    //端口号  
            namespace:'/sync'    //必须以“/”开头  
        };  
    }  
}
exports.Config = new config(); 
```

简单的工具类`util.js`
```javascript
function util(){  
    this.in_array = function(search,arr){  
        for(var i=0; i<arr.length; i++){  
            if(search==arr[i]){  
                return true;  
            }  
        }  
        return false;  
    };  
   
    this.array_search = function(search,arr){  
        for(var i=0; i<arr.length; i++){  
            if(search==arr[i]){  
                return i;  
            }  
        }  
        return -1;  
    }  
}  
   
exports.Util = new util();
```

接下来是web页面连接服务端

引入脚本

```markup
<script type="text/javascript" src="http://【node服务端地址】:【node端口】/socket.io/socket.io.js"></script>
```

使用
```javascript
var socket = io.connect('http://【node服务端地址】:【node服务端端口】/sync');  
socket.on('connect', function(){  
    console.log('connected to server');  
    socket.emit('message', 'bulabulabula...');  
});  
socket.on('disconnect', function(){  
    console.log('disconnected from server');  
});
```