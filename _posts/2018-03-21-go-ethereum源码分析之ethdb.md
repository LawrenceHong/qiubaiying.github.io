---
layout:     post
title:      go ethereum源码分析之ethdb
subtitle:  
date:       2018-03-21
author:     Leyou
header-img: img/ethereum.jpeg
catalog: true
tags:
---

> 这是第一次给大家分析以太坊的源码。因为今天面试了一家公司。人家让我对ethereum的一个模块分析下，写个report。我想想还是写个blog吧。以前用过levelDB，就从ethdb开始讲。

# 以太坊和levelDB
go-ethereum的所有的数据都存储在levelDB这个开源的Key-Value数据库中，整个区块链的所有数据都存储在一个levelDB的数据库中，levelDB可以按照文件大小进行切分文件，所以我们看到的区块链的数据都是一个一个小文件，这些小文件都是同一个levelDB实例。

# levelDB

## 特点(引用于官方)

key和value都是任意长度的字节数组;<br>
entry（即一条K-V记录）默认是按照key的字典顺序存储的，当然开发者也可以重载这个排序函数;<br>
提供的基本操作接口：Put()、Delete()、Get()、Batch();<br>
支持批量操作以原子操作进行;<br>
可以创建数据全景的snapshot(快照)，并允许在快照中查找数据;<br>
可以通过前向（或后向）迭代器遍历数据（迭代器会隐含的创建一个snapshot);<br>
自动使用Snappy压缩数据;<br>
可移植性;<br>

## 限制：

非关系型数据模型（NoSQL），不支持sql语句，也不支持索引；<br>
一次只允许一个进程访问一个特定的数据库；<br>
没有内置的C/S架构，但开发者可以使用LevelDB库自己封装一个server；<br>

# ethereum/ethdb目录下的源码分析.
首先源码很简单，就4个文件:
interface.go<br>
database.go<br>
database_test.go<br>
memory_database.go<br>

## interface.go
```
package ethdb

// Code using batches should try to add this much data to the batch.
// The value was determined empirically.
const IdealBatchSize = 100 * 1024

// Putter wraps the database write operation supported by both batches and regular databases.
//Putter接口定义了批量操作和普通操作的写入接口.
type Putter interface {
	Put(key []byte, value []byte) error
}

// Database wraps all database operations. All methods are safe for concurrent use.
//这里的所有借口都是线程安全的.
type Database interface {
	Putter
	Get(key []byte) ([]byte, error)
	Has(key []byte) (bool, error)
	Delete(key []byte) error
	Close()
	NewBatch() Batch
}

// Batch is a write-only database that commits changes to its host database
// when Write is called. Batch cannot be used concurrently.
//批量操作是一个只写数据库，它能将更改提交到其主数据库. 当写被调用，不能并发.
type Batch interface {
	Putter
	ValueSize() int // amount of data in the batch
	Write() error
	// Reset resets the batch for reuse
	Reset()
}
```
