---
title: "MongoDB Sharded Cluster - Config server"
date: 2019-12-12T17:24:51+09:00
---

지난 글에서 mongodb shard cluster 개념과 구성하는 방법에 대해 소개드렸고 
이번 글에서는 구성요소 중 하나인 config server에 대해 다루겠습니다.


### Config server 란?

config server는 shard cluster에서 필요한 모든 메타 정보들을 저장함  
메타정보에는 어느 chunk , 즉 어떤 data 가 어떤 shard 에 있는지, chunk 의 밸런싱 작업은 어떻게 해야할지 등의 메타정보와 사용자 인증 정보가 저장됨  
이렇게 중요한 정보들을 담고 있어서 가용성을 위해 하나의 replSet 으로 구성되어야 함  
* config server 정보
~~~
repl_conf:PRIMARY> use config;
switched to db config

repl_conf:PRIMARY> show collections;
actionlog
changelog
chunks
collections
databases
lockpings
locks
migrations
mongos
shards
system.sessions
tags
transactions
version 
~~~

* databases
~~~
repl_conf:PRIMARY> db.databases.find()
{ "_id" : "shard_test", "primary" : "repl_shard2", "partitioned" : false, "version" : { "uuid" : UUID("a2a1b712-6070-44c5-9370-ae711e9018bd"), "lastMod" : 1 } }

{ "_id" : "test", "primary" : "repl_shard2", "partitioned" : true, "version" : { "uuid" : UUID("5f6fa995-53c0-4677-a1ba-38aff579df77"), "lastMod" : 1 } }
~~~
=> shard cluster 가 가지고 있는 데이터베이스의 목록  
"partitioned" : 샤딩 활성화 여부 의미  


* collections
~~~
repl_conf:PRIMARY> db.collections.find()

{ "_id" : "test.testCollection2", "lastmodEpoch" : ObjectId("5d6288ab1a98ba4cb7e3aec2"), "lastmod" : ISODate("1970-02-19T17:02:47.299Z"), "dropped" : false, "key" : { "_id" : "hashed" }, "unique" : false, "uuid" : UUID("8cb727b3-81e2-4aff-a6a0-30b8f53240e4") }
~~~
=> sharding 된 collection 정보  
"_id" : 데이터베이스 이름+컬렉션 이름  
"key" : 컬렉션의 샤딩 키


* chunks

~~~
mongos> db.chunks.find()

{ "_id" : "test.testCollection2-_id_4611686018427387902", "ns" : "test.testCollection2", "min" : { "_id" : NumberLong("4611686018427387902") }, "max" : { "_id" : { "$maxKey" : 1 } }, "shard" : "repl_shard2", "lastmod" : Timestamp(1, 3), "lastmodEpoch" : ObjectId("5d6288ab1a98ba4cb7e3aec2"), "history" : [ { "validAfter" : Timestamp(1566738603, 6), "shard" : "repl_shard2" } ] }

{ "_id" : "test.testCollection2-_id_MinKey", "ns" : "test.testCollection2", "min" : { "_id" : { "$minKey" : 1 } }, "max" : { "_id" : NumberLong("-4611686018427387902") }, "shard" : "repl_shard1", "lastmod" : Timestamp(1, 0), "lastmodEpoch" : ObjectId("5d6288ab1a98ba4cb7e3aec2"), "history" : [ { "validAfter" : Timestamp(1566738603, 6), "shard" : "repl_shard1" } ] }

{ "_id" : "test.testCollection2-_id_-4611686018427387902", "ns" : "test.testCollection2", "min" : { "_id" : NumberLong("-4611686018427387902") }, "max" : { "_id" : NumberLong(0) }, "shard" : "repl_shard1", "lastmod" : Timestamp(1, 1), "lastmodEpoch" : ObjectId("5d6288ab1a98ba4cb7e3aec2"), "history" : [ { "validAfter" : Timestamp(1566738603, 6), "shard" : "repl_shard1" } ] }

{ "_id" : "test.testCollection2-_id_0", "ns" : "test.testCollection2", "min" : { "_id" : NumberLong(0) }, "max" : { "_id" : NumberLong("4611686018427387902") }, "shard" : "repl_shard2", "lastmod" : Timestamp(1, 2), "lastmodEpoch" : ObjectId("5d6288ab1a98ba4cb7e3aec2"), "history" : [ { "validAfter" : Timestamp(1566738603, 6), "shard" : "repl_shard2" } ] }
~~~
=> sharding 된 컬렉션의 모든 chunk 정보  
"_id" : database + collection + chunk가 갖는 값의 시작 값  
"shard" : 해당 chunk 가 속한 shard , 현재 각각 샤드에 두개씩의 chunk 가 존재함  


* shards

~~~
repl_conf:PRIMARY> db.shards.find()

{ "_id" : "repl_shard1", "host" : "repl_shard1/mongo_shard1:27018,mongo_shard2:27018,mongo_shard3:27018", "state" : 1 }
{ "_id" : "repl_shard2", "host" : "repl_shard2/mongo_shard4:27018,mongo_shard5:27018,mongo_shard6:27018", "state" : 1 }
~~~
=> shard cluster 내 등록된 모든 샤드 서버의 정보



* settings

~~~
{ "_id" : "chunksize", "value" : 64 }
{ "_id" : "balancer", "stopped" : false }
{ "_id" : "autosplit", "enabled" : true }
~~~

=> chunk size , 밸런싱 관련 설정  
초기 환경에서 db.settings.find() 를 수행해도 조회되는 게 없는데 이는 관련 설정이 저장된게 없기 때문  
아래와 같이 save command 수행 후엔 조회 가능  

~~~
repl_conf:PRIMARY> db.settings.save ( {_id:"chunksize",value:64} )
WriteResult({ "nMatched" : 0, "nUpserted" : 1, "nModified" : 0, "_id" : "chunksize" })

repl_conf:PRIMARY> db.settings.find()
{ "_id" : "chunksize", "value" : 64 }

repl_conf:PRIMARY> db.settings.save ( {_id:"autosplit","enabled":false} )

WriteResult({ "nMatched" : 0, "nUpserted" : 1, "nModified" : 0, "_id" : "autosplit" })
~~~

=> 위 설정은 mongos 의 sh.status() 와도 연동됨  
~~~
mongos> sh.status()

  autosplit:

Currently enabled: no
~~~

* chunk balancing 수행 시간 설정하기

~~~
repl_conf:PRIMARY> db.settings.update (
... {_id:"balancer"},
... {$set : {activeWindow: {start: "02:00", stop : "06:00" } } },
... {upsert : true } )

WriteResult({ "nMatched" : 0, "nUpserted" : 1, "nModified" : 0, "_id" : "balanver" })

repl_conf:PRIMARY> db.settings.find()
{ "_id" : "chunksize", "value" : 64 }
{ "_id" : "balancer", "stopped" : false }
{ "_id" : "autosplit", "enabled" : true }
{ "_id" : "balanver", "activeWindow" : { "start" : "02:00", "stop" : "06:00" } }
~~~
=> balancing 작업이 02:00 자동으로 수행되도록 설정  


* locks 

~~~
repl_conf:PRIMARY> db.locks.find()

{ "_id" : "config", "state" : 0, "process" : "ConfigServer", "ts" : ObjectId("5d6b7f15a9f5ecd49052a36f"), "when" : ISODate("2019-09-01T08:19:33.165Z"), "who" : "ConfigServer:conn164", "why" : "createCollection" }

{ "_id" : "config.system.sessions", "state" : 0, "process" : "ConfigServer", "ts" : ObjectId("5d6b7f15a9f5ecd49052a376"), "when" : ISODate("2019-09-01T08:19:33.172Z"), "who" : "ConfigServer:conn164", "why" : "createCollection" }

{ "_id" : "testdb", "state" : 0, "process" : "ConfigServer", "ts" : ObjectId("5d62889d3ed72a6b6729a5ca"), "when" : ISODate("2019-08-25T13:09:49.728Z"), "who" : "ConfigServer:conn24", "why" : "shardCollection" }

{ "_id" : "testdb.testCollection2", "state" : 0, "process" : "ConfigServer", "ts" : ObjectId("5d62889d3ed72a6b6729a5d1"), "when" : ISODate("2019-08-25T13:09:49.738Z"), "who" : "ConfigServer:conn24", "why" : "shardCollection" }

{ "_id" : "test.testCollection2", "state" : 0, "process" : "ConfigServer", "ts" : ObjectId("5d6288ab3ed72a6b6729a65a"), "when" : ISODate("2019-08-25T13:10:03.834Z"), "who" : "ConfigServer:conn24", "why" : "shardCollection" }
~~~
=> shard cluster 내 여러개의 mongos 가 동시에 동일한 chunk에 대한 작업을 시도하는 등의 이슈를 방지하기 위해  
작업을 수행하기 전 config server 의 locks collection의 lock을 획득 후에만 작업할 수 있도록 함


* changelog
~~~
repl_conf:PRIMARY> db.changelog.find()

{ "_id" : "69c6ea3e5527:27019-2019-09-01T08:43:10.827+0000-5d6b849ea9f5ecd49052d3ad", "server" : "69c6ea3e5527:27019", "shard" : "config", "clientAddr" : "111.111.0.8:58938", "time" : ISODate("2019-09-01T08:43:10.827Z"), 
"what" : "split", "ns" : "test.testCollection2", 
"details" : { "before" : { "min" : { "_id" : NumberLong("-4611686018427387902") }, "max" : { "_id" : NumberLong(0) }, "lastmod" : Timestamp(1, 3), "lastmodEpoch" : ObjectId("5d6288ab1a98ba4cb7e3aec2") }, "left" : { "min" : { "_id" : NumberLong("-4611686018427387902") }, "max" : { "_id" : NumberLong("-2315705090665887915") }, "lastmod" : Timestamp(1, 4), "lastmodEpoch" : ObjectId("5d6288ab1a98ba4cb7e3aec2") }, "right" : { "min" : { "_id" : NumberLong("-2315705090665887915") }, "max" : { "_id" : NumberLong(0) }, "lastmod" : Timestamp(1, 5), "lastmodEpoch" : ObjectId("5d6288ab1a98ba4cb7e3aec2") } } }
~~~

=> config 서버의 메타 정보 변경등이 기록됨  
ex) db, collection 생성, chunk split , migration 등,  위는 split 로그


* changelog 에서 split 로그만 확인하는 법 
~~~
repl_conf:PRIMARY> db.changelog.find( {"what":{$in: ["multi-split","split"]}}, {_id:0,what:1,ns:1,time:1}).sort({$natural:-1}).limit(5).pretty()

{

"time" : ISODate("2019-09-01T08:43:10.827Z"),

"what" : "split",

"ns" : "test.testCollection2"

}
~~~
