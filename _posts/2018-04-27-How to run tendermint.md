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

First, install dep:
```
$make get_tools
```
Maybe you will see the problem:
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
