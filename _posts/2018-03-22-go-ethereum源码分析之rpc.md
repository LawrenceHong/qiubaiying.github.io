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

> 这是第二次给大家分析以太坊的源码。昨天讲了个简单的ethdb, 今天讲个稍微难点的，RPC模块。你们如果读开源代码是不是经常会看到rpc目录，却又不知道是来干啥的，咋用的？我来给你们讲讲吧。。

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
# 读完代码后对，这个RPC的理解。
以太坊的代码都有注释，说实话，读完注释你就知道这个接口干啥的。<br>
对模块稍大点的先不要抠细节。<br>
我讲讲调用流程<br>
```
先是 cmd/geth/main.go 里
func geth(ctx *cli.Context) error {
	node := makeFullNode(ctx)
	startNode(ctx, node)
	node.Wait()
	return nil
}
调用了startNode, 传入context and 节点node.
然后在startNode里调用了utils.StartNode(stack), stack是node
func StartNode(stack *node.Node) {
	if err := stack.Start(); err != nil {     ///看这，相当于node 启动.
		Fatalf("Error starting protocol stack: %v", err)
	}
         ...
}
在这个node.go里的start函数:
func (n *Node) Start() error {
...
// Lastly start the configured RPC interfaces
	if err := n.startRPC(services); err != nil {  //这里启动RPC服务.
		for _, service := range services {
			service.Stop()
		}
		running.Stop()
		return err
	}
}
以下是startRPC的注释
// startRPC is a helper method to start all the various RPC endpoint during node
// startup. It's not meant to be called at any time afterwards as it makes certain
// assumptions about the state of the node.
往下跟进：调用了startInProc,
// startInProc initializes an in-process RPC endpoint.  看注释
func (n *Node) startInProc(apis []rpc.API) error {
	// Register all the APIs exposed by the services
	handler := rpc.NewServer()  //看到没，初始化RPC的server了。
	for _, api := range apis {
		if err := handler.RegisterName(api.Namespace, api.Service); err != nil {//这里就是注册了
			return err
		}
		n.log.Debug("InProc registered", "service", api.Service, "namespace", api.Namespace)
	}
	n.inprocHandler = handler
	return nil
}

//同理  这个startPRC    开启了不同的远端service的API// Start the various API endpoints, terminating all in case of errors   
所以想自己新加RPC可以从这里入手.
unc (n *Node) startRPC(services map[reflect.Type]Service) error {
	// Gather all the possible APIs to surface
	apis := n.apis()
	for _, service := range services {
		apis = append(apis, service.APIs()...)
	}
	// Start the various API endpoints, terminating all in case of errors   看这里
	if err := n.startInProc(apis); err != nil {
		return err
	}
	if err := n.startIPC(apis); err != nil {
		n.stopInProc()
		return err
	}
	if err := n.startHTTP(n.httpEndpoint, apis, n.config.HTTPModules, n.config.HTTPCors, n.config.HTTPVirtualHosts); err != nil {
		n.stopIPC()
		n.stopInProc()
		return err
	}
	if err := n.startWS(n.wsEndpoint, apis, n.config.WSModules, n.config.WSOrigins, n.config.WSExposeAll); err != nil {
		n.stopHTTP()
		n.stopIPC()
		n.stopInProc()
		return err
	}
	// All API endpoints started successfully
	n.rpcAPIs = apis
	return nil
}
```
# RPC使用举例
```
比如我们远端调用
curl --data '{"jsonrpc":"2.0","method":"eth_getBalance","params":["0x407", "latest"],"id":1}' localhost:8123
这是远端查询账户余额的, 走json rpc.
看代码node/node.go里的startHttp函数里，new的NewHTTPServer（rpc/http.go里）。
// ServeHTTP serves JSON-RPC requests over HTTP. 看注释. 
代码里new 了个Json解码器。
func (srv *Server) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	// Permit dumb empty requests for remote health-checks (AWS)
	if r.Method == http.MethodGet && r.ContentLength == 0 && r.URL.RawQuery == "" {
		return
	}
	if code, err := validateRequest(r); err != nil {
		http.Error(w, err.Error(), code)
		return
	}
	// All checks passed, create a codec that reads direct from the request body
	// untilEOF and writes the response to w and order the server to process a
	// single request.
	codec := NewJSONCodec(&httpReadWriteNopCloser{r.Body, w})
	defer codec.Close()

	w.Header().Set("content-type", contentType)
	srv.ServeSingleRequest(codec, OptionMethodInvocation)  //这里会调用到server.go 的serveRequest里
}
```
serveRequest 这个方法就是Server的主要处理流程。从codec读取请求，找到对应的方法并调用，然后把回应写入codec。<br>
```
// If a single shot request is executing, run and return immediately
		if singleShot {
			if batch {
				s.execBatch(ctx, codec, reqs)
			} else {
				s.exec(ctx, codec, reqs[0])
			}
			return nil
		}
```
理解了吧<br>
