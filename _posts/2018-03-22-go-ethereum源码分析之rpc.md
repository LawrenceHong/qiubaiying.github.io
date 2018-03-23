---
layout:     post
title:      go ethereum源码分析之rpc
subtitle:  
date:       2018-03-22
author:     Leyou
header-img: img/ethereum.jpeg
catalog: true
tags:
---

> 这是第二次给大家分析以太坊的源码。昨天讲了个简单的ethdb, 今天讲个稍微难点的，RPC模块。你们如果读开源代码是不是经常会看到rpc目录，却又不知道是来干啥的，咋用的？我来给你们讲讲吧。。用英文写blog得了，通用点

# JSON RPC API
//copy from https://www.51chain.net/portal/book/EthereumFrontierGuide/JSONRPCAPI-121.html
JSON is a lightweight data-interchange format. It can represent numbers, strings, ordered sequences of values, and collections of name/value pairs.

JSON-RPC is a stateless, light-weight remote procedure call (RPC) protocol. Primarily this specification defines several data structures and the rules around their processing. It is transport agnostic in that the concepts can be used within the same process, over sockets, over HTTP, or in many various message passing environments. It uses JSON (RFC 4627) as data format.

# RPC in ethereum
RPC go files in go-ethereum:<br>
![](https://raw.githubusercontent.com/LeyouHong/LeyouHong.github.io/master/img/rpc_code.jpg)<br>
I also read others' blog. Here is a structure picture of ethereum RPC:<br>
![](https://raw.githubusercontent.com/LeyouHong/LeyouHong.github.io/master/img/rpc_structure.png)<br>
So it's very clear that, we can list these code:<br>
```
client : client.go

server : server.go
         subscription.go
         
channels : websocket.go
           http.go
           ipc.go
           inproc
           
Json part :json.go

others : utils.go and types.go
```
Then we can read codes part by part.<br>

