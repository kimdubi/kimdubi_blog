---
title: "MongoDB Sharded Cluster"
date: 2019-12-11T18:21:34+09:00
---

sharding 은 데이터를 여러 서버에 분산해서 저장 및 처리할 수 있도록 하는 기술입니다.  
replicaset이 MongoDB의 고가용성을 위한 솔루션이라면, sharding은 분산 처리를 위한 솔루션이라 할 수 있으며  
Vertical, Horizontal 모두 적용 가능하나 일반적으로 하나의 Collection(table) 을 여러 샤드로 분산하는 Horizontal sharding 이 적용됩니다.  

이번 글에서는 MongoDB sharding 구성 방법을 살펴보고 다음 글에서 각각의 구성요소에 대해 좀 더 자세하게 살펴보도록 하겠습니다.  

### 아키텍처 개념

![](https://raw.githubusercontent.com/kimdubi/kimdubi.github.io/master/images/mongo_shard.png)

(https://dba.stackexchange.com/questions/82551/data-distribution-in-mongos-with-shards-or-replica-sets)

```
shard => 실제 데이터의 집합으로 샤드 키에 따라 shard1 / shard2 / shard3 에 데이터가 저장됨. 각각의 샤드 노드는 PSS (primary,secondary,secondary) / PSA (primary,secondary,arbiter) 형태의 replicaSet 으로 구성됨


config  => 샤드 서버에 저장된 데이터가 어떻게 나뉘어 분산돼 있는지에 대한 메타정보 및 사용자 정보 저장, 샤드 클러스터 내 컨피그 서버는 무조건 하나의 레플리카셋으로 구성되어야 함


mongos => 컨피그 서버의 메타정보를 읽어와 캐싱한 후 샤용자의 쿼리 요청을 어떤 샤드로 전달할지 정하고, 각 샤드로 부터 받은 쿼리 결과를 병합하여 리턴하는 역할
```



### 구성 예제

* Shard node

```
repl_shard1 => PSS 
repl_shard2 => PSS
```

* Config 

```
repl_conf => PSS
```

* Mongos

```
mongos 3대 
```


* config server 구성

```bash
docker run -d --name mongo_conf1 --network mongo-cluster mongo:4.1 mongod --replSet repl_conf --configsvr
docker run -d --name mongo_conf2 --network mongo-cluster mongo:4.1 mongod --replSet repl_conf --configsvr
docker run -d --name mongo_conf3 --network mongo-cluster mongo:4.1 mongod --replSet repl_conf --configsvr
```
=> shard cluster 내 config server 는 replSet 으로 이루어져야 하며 configsvr 옵션과 함께 기동되어야함

```
> rs.initiate({
... _id:"repl_conf",
... configsvr:true,
... members:[{_id:0,host:"mongo_conf1:27019"},{_id:1,host:"mongo_conf2:27019"},{_id:2,host:"mongo_conf3:27019"}] });
{
"ok" : 1,
"$gleStats" : {
"lastOpTime" : Timestamp(1566725886, 1),
"electionId" : ObjectId("000000000000000000000000")
},
"lastCommittedOpTime" : Timestamp(0, 0),
"$clusterTime" : {
"clusterTime" : Timestamp(1566725886, 1),
"signature" : {
"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
"keyId" : NumberLong(0)
}
},
"operationTime" : Timestamp(1566725886, 1)
}
```
=> config 서버 replSet 설정


* shard node #1 구성

```
dori:~ mac$ docker run -d  --net mongo-cluster --name mongo_shard1 mongo:4.1 mongod --replSet repl_shard1 --shardsvr
dori:~ mac$ docker run -d  --net mongo-cluster --name mongo_shard2 mongo:4.1 mongod --replSet repl_shard1 --shardsvr
dori:~ mac$ docker run -d  --net mongo-cluster --name mongo_shard3 mongo:4.1 mongod --replSet repl_shard1 --shardsvr
```
=> repl_shard1 라는 replSet으로 묶인 shard node 구성 

```
> rs.initiate({ _id:"repl_shard1", members:[ {_id:0,host:"mongo_shard1:27018"}, {_id:1,host:"mongo_shard2:27018"}, {_id:2,host:"mongo_shard3:27018"} ]} );
{
"ok" : 1,
"$clusterTime" : {
"clusterTime" : Timestamp(1566718019, 1),
"signature" : {
"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
"keyId" : NumberLong(0)
}
},
"operationTime" : Timestamp(1566718019, 1)
}
```

* shard node #2 구성

```
dori:~ mac$ docker run -d  --net mongo-cluster --name mongo_shard4 mongo:4.1 mongod --replSet repl_shard2 --shardsvr
dori:~ mac$ docker run -d  --net mongo-cluster --name mongo_shard5 mongo:4.1 mongod --replSet repl_shard2 --shardsvr
dori:~ mac$ docker run -d  --net mongo-cluster --name mongo_shard6 mongo:4.1 mongod --replSet repl_shard2 --shardsvr
```
=> repl_shard2 라는 replSet으로 묶인 shard node 구성 


```
> rs.initiate( {_id:"repl_shard2", members:[ {_id:0,host:"mongo_shard4:27018"}, {_id:1,host:"mongo_shard5:27018"}, {_id:2,host:"mongo_shard6:27018"}]} ) ;
{
"ok" : 1,
"$clusterTime" : {
"clusterTime" : Timestamp(1566719750, 1),
"signature" : {
"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
"keyId" : NumberLong(0)
}
},
"operationTime" : Timestamp(1566719750, 1)
}
```



* mongos 구성

```
docker run -d --network mongo-cluster --name mongos1 -p 27017:27017 mongo:4.1 mongos --configdb repl_conf/mongo_conf1:27019,mongo_conf2:27019,mongo_conf3:27019 --bind_ip_all

docker run -d --network mongo-cluster --name mongos2 -p 27027:27017 mongo:4.1 mongos --configdb repl_conf/mongo_conf1:27019,mongo_conf2:27019,mongo_conf3:27019 --bind_ip_all

docker run -d --network mongo-cluster --name mongos3 -p 27037:27017 mongo:4.1 mongos --configdb repl_conf/mongo_conf1:27019,mongo_conf2:27019,mongo_conf3:27019 --bind_ip_all
```

=> mongos 는 config server 의 메타 정보를 읽어와 client 의 request 를 처리할 shard 를 정하기 때문에 mongos 가 읽을 config server 를 정의해야함

```
sh.addShard("repl_shard1/mongo_shard1:27018,mongo_shard2:27018,mongo_shard3:27018")
sh.addShard("repl_shard2/mongo_shard4:27018,mongo_shard5:27018,mongo_shard6:27018")
```
=> shard node #1 과 #2 를 추가함  

```
mongos> sh.addShard("repl_shard1/mongo_shard1:27018,mongo_shard2:27018,mongo_shard3:27018")
{
"shardAdded" : "repl_shard1",
"ok" : 1,
"operationTime" : Timestamp(1566738338, 4),
"$clusterTime" : {
"clusterTime" : Timestamp(1566738338, 4),
"signature" : {
"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
"keyId" : NumberLong(0)
}
}
}

mongos> sh.addShard("repl_shard2/mongo_shard4:27018,mongo_shard5:27018,mongo_shard6:27018")
{
"shardAdded" : "repl_shard2",
"ok" : 1,
"operationTime" : Timestamp(1566738339, 8),
"$clusterTime" : {
"clusterTime" : Timestamp(1566738339, 8),
"signature" : {
"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
"keyId" : NumberLong(0)
}
}
}
```

### shard 구성 확인

```
mongos> sh.status()
--- Sharding Status ---
  sharding version: {
   "_id" : 1,
   "minCompatibleVersion" : 5,
   "currentVersion" : 6,
   "clusterId" : ObjectId("5d62570b3ed72a6b6728f2f3")
  }
  shards:
{  "_id" : "repl_shard1",  "host" : "repl_shard1/mongo_shard1:27018,mongo_shard2:27018,mongo_shard3:27018",  "state" : 1 }
{  "_id" : "repl_shard2",  "host" : "repl_shard2/mongo_shard4:27018,mongo_shard5:27018,mongo_shard6:27018",  "state" : 1 }
  active mongoses:
"4.1.13" : 3
  autosplit:
Currently enabled: yes
  balancer:
Currently enabled:  yes
Currently running:  no
Failed balancer rounds in last 5 attempts:  0
Migration Results for the last 24 hours:
No recent migrations
  databases:
{  "_id" : "config",  "primary" : "config",  "partitioned" : true }
```
=> shard node : repl_shard1 , repl_shard2  mongos : 3대 기동중


autosplit : mongodb에서는 sharding 의 기준인 collection 을 여러 조각으로 파티션하고 각 조각을 여러 샤드 서버에 분산해서 저장하는데 이 조각을 chunk 라고 함.

chunk 는 default 64MB 까지 커질 수 있는데 이 때 이 chunk를 자동으로 split 하겠다는 의미

balancer : chunk 의 split 이 빈번하면 하나의 shard node에만 chunk 수가 많아지고 밸런스가 깨질 수 있는데 밸런스를 맞추기 위해

chunk를 다른 샤드 노드로 migration 하는 역할
