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

# levelDB(引用于官方)

## 特点

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

# ethereum/ethdb目录下的源码分析(我也是参考了下别人，看看后不难的，哈哈，捡个漏).
首先源码很简单，就4个文件:<br>
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
## memory_database.go
```
//memory database, 顾名思义， 你看，用了map吧。还用了把读写多线程锁保护呢。
type MemDatabase struct {
	db   map[string][]byte
	lock sync.RWMutex
}

func NewMemDatabase() (*MemDatabase, error) {
	return &MemDatabase{
		db: make(map[string][]byte),
	}, nil
}

func (db *MemDatabase) Put(key []byte, value []byte) error {
	db.lock.Lock()
	defer db.lock.Unlock()

	db.db[string(key)] = common.CopyBytes(value)
	return nil
}

func (db *MemDatabase) Has(key []byte) (bool, error) {
	db.lock.RLock()
	defer db.lock.RUnlock()

	_, ok := db.db[string(key)]
	return ok, nil
}

func (db *MemDatabase) Get(key []byte) ([]byte, error) {
	db.lock.RLock()
	defer db.lock.RUnlock()

	if entry, ok := db.db[string(key)]; ok {
		return common.CopyBytes(entry), nil
	}
	return nil, errors.New("not found")
}

func (db *MemDatabase) Keys() [][]byte {
	db.lock.RLock()
	defer db.lock.RUnlock()

	keys := [][]byte{}
	for key := range db.db {
		keys = append(keys, []byte(key))
	}
	return keys
}

func (db *MemDatabase) Delete(key []byte) error {
	db.lock.Lock()
	defer db.lock.Unlock()

	delete(db.db, string(key))
	return nil
}
```
继续看Batch: 哎，就是批量操作而已，一看便知。
```
type kv struct{ k, v []byte }

type memBatch struct {
	db     *MemDatabase
	writes []kv
	size   int
}

func (b *memBatch) Put(key, value []byte) error {
	b.writes = append(b.writes, kv{common.CopyBytes(key), common.CopyBytes(value)})
	b.size += len(value)
	return nil
}

func (b *memBatch) Write() error {
	b.db.lock.Lock()
	defer b.db.lock.Unlock()

	for _, kv := range b.writes {
		b.db.db[string(kv.k)] = kv.v
	}
	return nil
}
```
## database.go
```
//看import，是对go level进行了封装.
import (
	"strconv"
	"strings"
	"sync"
	"time"

	"github.com/ethereum/go-ethereum/log"
	"github.com/ethereum/go-ethereum/metrics"
	"github.com/syndtr/goleveldb/leveldb"
	"github.com/syndtr/goleveldb/leveldb/errors"
	"github.com/syndtr/goleveldb/leveldb/filter"
	"github.com/syndtr/goleveldb/leveldb/iterator"
	"github.com/syndtr/goleveldb/leveldb/opt"
)

//看这个结构，用了github.com/syndtr/goleveldb/leveldb的leveldb的封装.
//Mertrics是记录了数据库的使用情况吧。
type LDBDatabase struct {
	fn string      // filename for reporting
	db *leveldb.DB // LevelDB instance

	compTimeMeter  metrics.Meter // Meter for measuring the total time spent in database compaction
	compReadMeter  metrics.Meter // Meter for measuring the data read during compaction
	compWriteMeter metrics.Meter // Meter for measuring the data written during compaction
	diskReadMeter  metrics.Meter // Meter for measuring the effective amount of data read
	diskWriteMeter metrics.Meter // Meter for measuring the effective amount of data written

	quitLock sync.Mutex      // Mutex protecting the quit channel access
	quitChan chan chan error // Quit channel to stop the metrics collection before closing the database

	log log.Logger // Contextual logger tracking the database path
}

//往下看看Put,Has, 没用锁，查阅了下资料，github.com/syndtr/goleveldb/leveldb封装之后是支持多线程，可并发。
// Put puts the given key / value to the queue
func (db *LDBDatabase) Put(key []byte, value []byte) error {
	// Generate the data to write to disk, update the meter and write
	//value = rle.Compress(value)

	return db.db.Put(key, value, nil)
}

func (db *LDBDatabase) Has(key []byte) (bool, error) {
	return db.db.Has(key, nil)
}

//NewLDBDatabase时候没有初始化Mertrics，在Meter里初始化的。
//这个方法每3秒钟获取一次leveldb内部的计数器，然后把他们公布到metrics子系统。
//不停的循环， 除非quitChan收到了一个退出信号。就是不停打印leveldb的使用情况咯。
func (db *LDBDatabase) Meter(prefix string) {
	// Short circuit metering if the metrics system is disabled
	if !metrics.Enabled {
		return
	}
	// Initialize all the metrics collector at the requested prefix
	db.getTimer = metrics.NewTimer(prefix + "user/gets")
	db.putTimer = metrics.NewTimer(prefix + "user/puts")
	db.delTimer = metrics.NewTimer(prefix + "user/dels")
	db.missMeter = metrics.NewMeter(prefix + "user/misses")
	db.readMeter = metrics.NewMeter(prefix + "user/reads")
	db.writeMeter = metrics.NewMeter(prefix + "user/writes")
	db.compTimeMeter = metrics.NewMeter(prefix + "compact/time")
	db.compReadMeter = metrics.NewMeter(prefix + "compact/input")
	db.compWriteMeter = metrics.NewMeter(prefix + "compact/output")

	// Create a quit channel for the periodic collector and run it
	db.quitLock.Lock()
	db.quitChan = make(chan chan error)
	db.quitLock.Unlock()

	go db.meter(3 * time.Second)
}

//继续往下走, 下面的注释是我们调用 db.db.GetProperty("leveldb.stats")返回的字符串，后续的代码需要解析这个字符串并把信息写入到Meter中。<br>
//具体细节不分析了>_<
// meter periodically retrieves internal leveldb counters and reports them to
// the metrics subsystem.
// This is how a stats table look like (currently).

//   Compactions
//    Level |   Tables   |    Size(MB)   |    Time(sec)  |    Read(MB)   |   Write(MB)
//   -------+------------+---------------+---------------+---------------+---------------
//      0   |          0 |       0.00000 |       1.27969 |       0.00000 |      12.31098
//      1   |         85 |     109.27913 |      28.09293 |     213.92493 |     214.26294
//      2   |        523 |    1000.37159 |       7.26059 |      66.86342 |      66.77884
//      3   |        570 |    1113.18458 |       0.00000 |       0.00000 |       0.00000

func (db *LDBDatabase) meter(refresh time.Duration) {
	// Create the counters to store current and previous values
	counters := make([][]float64, 2)
	for i := 0; i < 2; i++ {
		counters[i] = make([]float64, 3)
	}
	// Iterate ad infinitum and collect the stats
	for i := 1; ; i++ {
		// Retrieve the database stats
		stats, err := db.db.GetProperty("leveldb.stats")
		if err != nil {
			db.log.Error("Failed to read database stats", "err", err)
			return
		}
		// Find the compaction table, skip the header
		lines := strings.Split(stats, "\n")
		for len(lines) > 0 && strings.TrimSpace(lines[0]) != "Compactions" {
			lines = lines[1:]
		}
		if len(lines) <= 3 {
			db.log.Error("Compaction table not found")
			return
		}
		lines = lines[3:]

		// Iterate over all the table rows, and accumulate the entries
		for j := 0; j < len(counters[i%2]); j++ {
			counters[i%2][j] = 0
		}
		for _, line := range lines {
			parts := strings.Split(line, "|")
			if len(parts) != 6 {
				break
			}
			for idx, counter := range parts[3:] {
				value, err := strconv.ParseFloat(strings.TrimSpace(counter), 64)
				if err != nil {
					db.log.Error("Compaction entry parsing failed", "err", err)
					return
				}
				counters[i%2][idx] += value
			}
		}
		// Update all the requested meters
		if db.compTimeMeter != nil {
			db.compTimeMeter.Mark(int64((counters[i%2][0] - counters[(i-1)%2][0]) * 1000 * 1000 * 1000))
		}
		if db.compReadMeter != nil {
			db.compReadMeter.Mark(int64((counters[i%2][1] - counters[(i-1)%2][1]) * 1024 * 1024))
		}
		if db.compWriteMeter != nil {
			db.compWriteMeter.Mark(int64((counters[i%2][2] - counters[(i-1)%2][2]) * 1024 * 1024))
		}
		// Sleep a bit, then repeat the stats collection
		select {
		case errc := <-db.quitChan:
			// Quit requesting, stop hammering the database
			errc <- nil
			return

		case <-time.After(refresh):
			// Timeout, gather a new set of stats
		}
	}
}
```
# ethdb在以太坊的使用例子:
看这个CreatDB, 这里用到了db.Meter("eth/db/chaindata/"). 传了个prefix，就是来打印数据链的在leveldb上的信息的。
![](https://raw.githubusercontent.com/LeyouHong/LeyouHong.github.io/master/img/ethdb_usage3.jpg)

# 我自己对这个模块的测试
//明天更。。。
