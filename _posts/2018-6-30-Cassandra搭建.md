---
layout:     post
title:      How to use Cassandra
subtitle:  
date:       2018-06-30
author:     Leyou
header-img: img/cassandra.jpg
catalog: true
tags:   
---
> How to install Cassandra(3.11.2).

# Install it in Mint 18
Please refer to this page : [install](http://cassandra.apache.org/doc/latest/getting_started/installing.html)</br>
I install cassandra from Debian package.

# start Cassandra
If you want to use default config. Please add this two commands before start it
```
sudo chown -R $USER:$GROUP /var/log/cassandra
sudo chown -R $USER:$GROUP /var/lib/cassandra
```
run it 
```
cassandra -f
```
