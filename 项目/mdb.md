---
catalog: projects
tags: [mongodb, mgo, go]
github: https://github.com/ti/mdb
link: https://github.com/ti/mdb
date: 2017-05-05
picture: img/mongodb.jpg
---

# mdb,一个以db为单位，带重连机制的mgo封装

Golang 一直缺失官方的Mongodb库， 对于Go开发者，普遍采用mgo库作为go语言的mongodb驱动库，如果你使用过，你可能在每次查询数据库时习惯了以下的用法：

```go
copy := session.Clone; 
defter copy.Close();
copy.DB("dbname).C("col").Find(...)
```

这样的用法虽然丑陋，但是它解决了我们使用mongodb最大的坑，“当MongoDB断开后，mongodb无法重连" 的问题， 但是这样的用法也会造成一些问题：

* 每次查询mongodb库，都会克隆一个新的session，并刷新，造成资源浪费。
* 当你的程序在高并发时，你会发现你的mongodb链接达到了 上百和上千个。
* 高并发时，大概会有30%的请求率，会返回 "Closed explicitly" 或者 "EOF"

为了解决这个问题，我重新封装了mgo库，叫做mdb，自带重连机制，使用方法也和普通的SQL库有几分相似

```go
//初始化db对象，db_for_connect 一般为admin， db 为默认数据库名
db, err := mdb.Dial("mongodb://username:password@127.0.0.1:27017/db_for_connect?db=test")
//向db的people 表添加数据
db.C("people").Insert(&Person{"Ale", "+55 53 8116 9639"})
```

这样操作会有以下优点：

1. 简化monogdb操作的代码量。在业务层，你只需`db.C("people").Insert(...)`，来进行数据操作。
2. 减少长连数量，并提高稳定性。一般情况下，你和mongodb只有一个长连，随着业务量的增加可能会增加2个，3个或更多，这些长连会自我维护和释放。


github地址: https://github.com/ti/mdb

详情如下：


# mdb

A rich mongodb driver based on mgo and auto refresh when "Closed explicitly" or "EOF"

# feature

* do not need `copy := session.Clone; defter copy.Close();`
* use db instance in project
* less tcp connections
* auto refresh connections when connection is break
* more simple

# why this one

if you use  `copy := session.Clone; defter copy.Close(); copy.DB("dbname).C("col").Find(...)` 

you may got "Closed explicitly" or "EOF"  when in high concurrency

# quick start

```go
type Person struct {
	Name string
	Phone string
}

func main() {
    //the test is default db
	db, err := mdb.Dial("mongodb://127.0.0.1:27017/test")
	if err != nil {
		panic(err)
	}
	defer db.Close()
  
	c := db.C("people")
	err = c.Insert(&Person{"Ale", "+55 53 8116 9639"},&Person{"Cla", "+55 53 8402 8510"})
	if err != nil {
		log.Fatal(err)
	}
	
	result := Person{}
	err = c.Find(bson.M{"name": "Ale"}).One(&result)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println("Phone:", result.Phone)
}
```
# when mongo connection string and database name is different?

use [mongo-connection-string](https://docs.mongodb.com/manual/reference/connection-string/) + `&db={db_name}` use  to config your db name

example:

```go
db, err := mdb.Dial("mongodb://username:password@192.168.31.5:27017?db=test")
```

when username is not an administrator

```go
db, err := mdb.Dial("mongodb://username:password@127.0.0.1:27017/test")
//when you have to connect another db first
db, err := mdb.Dial("mongodb://username:password@127.0.0.1:27017/db_for_connect?db=test")
```

# new connection string parameter

maxRetries  : max retries time  when network is error, default is 2

db          : database name when your connection string and database name is different

FULL Example:

```go
db, err := mdb.Dial("mongodb://username:password@127.0.0.1:27017?db=test&maxRetries=2")

```








