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
Please refer to this page : [install](http://cassandra.apache.org/doc/latest/getting_started/installing.html)<br>
I install cassandra from Debian package.

# start Cassandra
If you want to use default config. Please add this two commands before start it
```
sudo chown -R $USER:$GROUP /var/log/cassandra
sudo chown -R $USER:$GROUP /var/lib/cassandra
```
run it 
```
cassandra -f &
```
If you start cassandra failed, maybe the port has already be used. So please use this
```
ps -aux | grep cass
```
You can get cassandra PID and use the command to kill it.
```
kill -9 ${PID}
```
# check nodes status
```
parallels@linuxmint-vm ~ $ nodetool status
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address    Load       Tokens       Owns (effective)  Host ID                               Rack
UN  127.0.0.1  204.71 KiB  256          100.0%            75317cee-c6aa-487a-9040-bd7e3c3d92aa  rack1
```
```
$ nodetool info
```
# How to use CQL
[CQL DOC](https://www.tutorialspoint.com/cassandra/cassandra_create_keyspace.htm)
