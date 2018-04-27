---
layout:     post
title:      Tendermint App
subtitle:  
date:       2018-04-27
author:     Leyou
header-img: img/ethereum.jpeg
catalog: true
tags:   
---

> If you didn't install the tendermint. Please refer to my previous blog.

# Run the tendermint
make sure you have already install the tendermint.
```
tendermint init
tendermint node --proxy_app=kvstore
```

# First Tendermint App
## Install abci
If you didn't download abci, please use this cmd to download it.
Please make sure you set a correct $GOPATH
```
go get -u github.com/tendermint/abci/cmd/abci-cli
```
install abci
```
cd $GOPATH/src/github.com/tendermint/abci
make get_tools
make get_vendor_deps
make install
```
## KVStore - A First Example
```
$ abci-cli kvstore
I[04-27|21:45:47.286] Starting ABCIServer                          module=abci-server impl=ABCIServer
I[04-27|21:45:47.286] Waiting for new connection...                module=abci-server 
```
Then we start tendermint
```
$tendermint init
$tendermint node

If you have used Tendermint, you may want to reset the data for a new blockchain by running tendermint unsafe_reset_all. 
Then you can run tendermint node to start Tendermint, and connect to the app. 
```
You should see Tendermint making blocks! We can get the status of our Tendermint node as follows:
```
curl -s localhost:46657/status
```
The -s just silences curl. For nicer output, pipe the result into a tool like jq or jsonpp.

Now let’s send some transactions to the kvstore.
```
curl -s 'localhost:46657/broadcast_tx_commit?tx="abcd"'
```
Note the single quote (') around the url, which ensures that the double quotes (") 
are not escaped by bash. This command sent a transaction with bytes abcd, so abcd
will be stored as both the key and the value in the Merkle tree. The response should look something like:
```
{
  "jsonrpc": "2.0",
  "id": "",
  "result": {
    "check_tx": {
      "code": 0,
      "data": "",
      "log": ""
    },
    "deliver_tx": {
      "code": 0,
      "data": "",
      "log": ""
    },
    "hash": "2B8EC32BA2579B3B8606E42C06DE2F7AFA2556EF",
    "height": 154
  }
}
```
