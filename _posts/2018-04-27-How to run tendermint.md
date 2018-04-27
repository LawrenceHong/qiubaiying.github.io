---
layout:     post
title:      How to install Tendermint
subtitle:  
date:       2018-04-27
author:     Leyou
header-img: img/ethereum.jpeg
catalog: true
tags:   
---

> I want to use tendermint. I will introduces how to install tendermint today.

# Get the source code.
```
$go get github.com/tendermint/tendermint/cmd/tendermint
```
If the installation failed, a dependency may have been updated and become incompatible with the latest Tendermint master branch. We solve this using the dep tool for dependency management.

# Install
First, install dep:
```
$make get_tools
```
## Trouble shooting
Maybe you will see these problem:

### failed with gometalinter.v2
```
make: gometalinter.v2: Command not found
```
It seems that we need to install gometalinter : https://github.com/alecthomas/gometalinter
We can use this cmd to install
```
$go get -u gopkg.in/alecthomas/gometalinter.v2
```
But we will still failed. Let's see Makefile.
```
get_tools:
        @echo "--> Installing tools"
        go get -u -v $(GOTOOLS)
        @gometalinter.v2 --install
        // Here we try to use gometalinter.v2 cmd in Linux, but we only have source code of gometalinter.v2.
```
Let's build it
```
cd $GOPATH/src/gopkg.in/alecthomas/gometalinter.v2
go build

if you still failed, check the PATH
make sure like it :
export PATH=$PATH:$GOROOT/bin:$GOPATH/bin
```
### The GOPATH of govet is wrong 
``` 
get "gopkg.in/alecthomas/gometalinter.v2/vendor/gopkg.in/yaml.v2": verifying non-authoritative meta tag
...
can't load package: package github.com/dnephin/govet: cannot find package "github.com/dnephin/govet" in any of:
	$GOROOT/src/github.com/dnephin/govet (from $GOROOT)
	$GOPATH/src/github.com/alecthomas/gometalinter/_linters/src/github.com/dnephin/govet (from $GOPATH)
```
Because some issues. We have download govet manually.
```
$mkdir $GOPATH/src/github.com/alecthomas/gometalinter/_linters/github.com/dnephin
$cd $GOPATH/src/github.com/alecthomas/gometalinter/_linters/github.com/dnephin
$git clone https://github.com/dnephin/govet

After this step. We can go back to tendermint and do 
$make get_tools 

You will pass
```
