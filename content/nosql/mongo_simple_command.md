---
title: "MongoDB simple command"
date: 2019-12-07T17:19:56+09:00
---



### Mongo DB 접속      

```
dori:bin mac$ mongo

MongoDB shell version v4.0.9
connecting to: mongodb://127.0.0.1:27017/?gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("97c63b7c-e54a-447f-9507-c775b4986954") }
MongoDB server version: 4.0.9
Server has startup warnings:
2019-05-11T16:07:53.047+0900 I CONTROL  [initandlisten]
2019-05-11T16:07:53.047+0900 I CONTROL  [initandlisten] ** WARNING: Access control is not enabled for the database.
2019-05-11T16:07:53.048+0900 I CONTROL  [initandlisten] **          Read and write access to data and configuration is unrestricted.
2019-05-11T16:07:53.049+0900 I CONTROL  [initandlisten]
---
Enable MongoDB's free cloud-based monitoring service, which will then receive and display
metrics about your deployment (disk utilization, CPU, operation statistics, etc).
The monitoring data will be available on a MongoDB website with a unique URL accessible to you
and anyone you share the URL with. MongoDB may use this information to make product
improvements and to suggest MongoDB products and deployment options to you.

To enable free monitoring, run the following command: db.enableFreeMonitoring()
To permanently disable this reminder, run the following command: db.disableFreeMonitoring()
```


### create database  


```
> show dbs;
admin   0.000GB
config  0.000GB
local   0.000GB

> use test;
switched to db test

> show dbs;
admin   0.000GB
config  0.000GB
local   0.000GB
```
* test db가 안보이는 이유는 test db 에 아무런 collection 이 없기 때문 collection 이 있는 db만 조회됨

### create collection  

```
> db
test

> test.TestCollection.insertOne(
... {name:"kimdubi", age:29, interest: ["Infra","DB","NoSQL"]})

2019-05-11T16:36:13.481+0900 E QUERY    [js] ReferenceError: test is not defined :
@(shell):1:1

>
> db.TestCollection.insertOne(
... {name:"kimdubi", age:29, interest: ["Infra","DB","NoSQL"]})
{
"acknowledged" : true,
"insertedId" : ObjectId("5cd67b82d7c83919d5d619fa")
}

> show collections
TestCollection

> db.createCollection("TestCollection_2")
{ "ok" : 1 }

> show collections;
TestCollection
TestCollection_2
```
* db 커맨드는 현재 사용중인 database 확인용  
dml 커맨드는 db.컬렉션네임.커맨드 로 수행됨
* 데이터를 넣으며 테이블 생성하는 방법과 넣지 않고 명시적으로 테이블 create 할 수 있음

### insert 

* 한 건 씩 혹은 여러건씩 insert 가능  
```
> db.TestCollection.insertOne(
{name:"kimdubi",age:29,interset: ["Infra","DB","NoSQL"]})

> db.TestCollection.insertMany([
... {name:"kimdubi",age:29,interset: ["Infra","DB","NoSQL"]},
... {name:"kimdubi",age:29,interset: ["Infra","DB","NoSQL"]},
... {name:"kimdubi",age:29,interset: ["Infra","DB","NoSQL"]},
... {name:"kimdubi",age:29,interset: ["Infra","DB","NoSQL"]},
... {name:"kimdubi2",age:39,interset: ["Infra","Cloud","AWS"]},
... {name:"kimdubi3",age:49,interset: ["Programming","Frontend","nodejs"]}
... ])
{
"acknowledged" : true,
"insertedIds" : [
ObjectId("5cd68b1ed7c83919d5d61a02"),
ObjectId("5cd68b1ed7c83919d5d61a03"),
ObjectId("5cd68b1ed7c83919d5d61a04"),
ObjectId("5cd68b1ed7c83919d5d61a05"),
ObjectId("5cd68b1ed7c83919d5d61a06"),
ObjectId("5cd68b1ed7c83919d5d61a07")
]
}
```

*  bulk 로 많은 dml 커맨드를 한번에 수행 할 수 있음
``` 
> var users=
... [
... {name:"kimdubi",age:29,interset: ["Infra","DB","NoSQL"]},
... {name:"kimdubi",age:29,interset: ["Infra","DB","NoSQL"]},
... {name:"kimdubi",age:29,interset: ["Infra","DB","NoSQL"]},
... {name:"kimdubi",age:29,interset: ["Infra","DB","NoSQL"]},
... {name:"kimdubi2",age:39,interset: ["Infra","Cloud","AWS"]},
... {name:"kimdubi3",age:49,interset: ["Programming","Frontend","nodejs"]}
... ];
>
>
> db.TestCollection.insert(users);
BulkWriteResult({
"writeErrors" : [ ],
"writeConcernErrors" : [ ],
"nInserted" : 6,
"nUpserted" : 0,
"nMatched" : 0,
"nModified" : 0,
"nRemoved" : 0,
"upserted" : [ ]
})
```

### select

* forEach(printjson)  / pretty() 로 보기 좋게 출력
```  
> db.TestCollection.find()
{ "_id" : ObjectId("5cd67b82d7c83919d5d619fa"), "name" : "kimdubi", "age" : 29, "interest" : [ "Infra", "DB", "NoSQL" ] } .....

     
> db.TestCollection.find().forEach(printjson);
{
"_id" : ObjectId("5cd67b82d7c83919d5d619fa"),
"name" : "kimdubi",
"age" : 29,
"interest" : [
"Infra",
"DB",
"NoSQL"
]
}
{
"_id" : ObjectId("5cd68655d7c83919d5d619fb"),
"name" : "kimdubi",
"age" : 29,
"interset" : [
"Infra",
"DB",
"NoSQL"
]
}
.
.

> db.TestCollection.find().pretty();
{
"_id" : ObjectId("5cd67b82d7c83919d5d619fa"),
"name" : "kimdubi",
"age" : 29,
"interest" : [
"Infra",
"DB",
"NoSQL"
]
}

> db.TestCollection.find( {name: "kimdubi3"}).pretty()
{
"_id" : ObjectId("5cd6895dd7c83919d5d61a01"),
"name" : "kimdubi3",
"age" : 49,
"interset" : [
"Programming",
"Frontend",
"nodejs"
]
}
```

* where 조건
```
> db.TestCollection.find( { "age": {$gt:32}} ).pretty()

{
"_id" : ObjectId("5cd6895dd7c83919d5d61a00"),
"name" : "kimdubi2",
"age" : 39,
"interset" : [
"Infra",
"Cloud",
"AWS"
]
}
{
"_id" : ObjectId("5cd6895dd7c83919d5d61a01"),
"name" : "kimdubi3",
"age" : 49,
"interset" : [
"Programming",
"Frontend",
"nodejs"
]
}
```


* 특정 필드만 조회 및 정렬 
```
> db.TestCollection.find( {} , { "name":1}).sort({"age":1})

{ "_id" : ObjectId("5cd67b82d7c83919d5d619fa"), "name" : "kimdubi" }
{ "_id" : ObjectId("5cd68655d7c83919d5d619fb"), "name" : "kimdubi" }
{ "_id" : ObjectId("5cd6895dd7c83919d5d619fc"), "name" : "kimdubi" }
{ "_id" : ObjectId("5cd6895dd7c83919d5d619fd"), "name" : "kimdubi" }
{ "_id" : ObjectId("5cd6895dd7c83919d5d619fe"), "name" : "kimdubi" }
{ "_id" : ObjectId("5cd6895dd7c83919d5d619ff"), "name" : "kimdubi" }
{ "_id" : ObjectId("5cd6895dd7c83919d5d61a00"), "name" : "kimdubi2" }
{ "_id" : ObjectId("5cd6895dd7c83919d5d61a01"), "name" : "kimdubi3" }
```

```
> db.TestCollection.find({}, {"name":1,"interset":1} )

{ "_id" : ObjectId("5cd67b82d7c83919d5d619fa"), "name" : "kimdubi" }
{ "_id" : ObjectId("5cd68655d7c83919d5d619fb"), "name" : "kimdubi", "interset" : [ "Infra", "DB", "NoSQL" ] }
{ "_id" : ObjectId("5cd6895dd7c83919d5d619fc"), "name" : "kimdubi", "interset" : [ "Infra", "DB", "NoSQL" ] }
{ "_id" : ObjectId("5cd6895dd7c83919d5d619fd"), "name" : "kimdubi", "interset" : [ "Infra", "DB", "NoSQL" ] }
```
* 컬럼명:0 (default) => 조회안함 / 컬럼명:1 =>조회

### update

``` 
> db.TestCollection.update( {"name":"kimdubi"}, {$set:{"name":"super kimdubi"}}, {multi:true} )
WriteResult({ "nMatched" : 10, "nUpserted" : 0, "nModified" : 10 })

     
>
> db.TestCollection.find()
{ "_id" : ObjectId("5cd67b82d7c83919d5d619fa"), "name" : "super kimdubi", "age" : 29, "interest" : [ "Infra", "DB", "NoSQL" ] }
{ "_id" : ObjectId("5cd68655d7c83919d5d619fb"), "name" : "super kimdubi", "age" : 29, "interset" : [ "Infra", "DB", "NoSQL" ] }
{ "_id" : ObjectId("5cd6895dd7c83919d5d619fc"), "name" : "super kimdubi", "age" : 29, "interset" : [ "Infra", "DB", "NoSQL" ] }
{ "_id" : ObjectId("5cd6895dd7c83919d5d619fd"), "name" : "super kimdubi", "age" : 29, "interset" : [ "Infra", "DB", "NoSQL" ] }
```
* 기본적으로 MongoDB에서는 한건씩 업데이트를 하는데 {multi:true} 옵션을 명시하면 여러건 동시에 업데이트 가능

### delete

* 조건에 맞는 다큐먼트 1건 지우기
``` 
> db.TestCollection.remove({ "name": "super kimdubi"} )
WriteResult({ "nRemoved" : 10 })
```

* 여러건 지우기
```
> db.TestCollection.remove ({ "name":"kimdubi"}, 2 )
WriteResult({ "nRemoved" : 0 })
```

* 전부 지우기
```
> db.TestCollection.remove( {} )
WriteResult({ "nRemoved" : 4 })
```
