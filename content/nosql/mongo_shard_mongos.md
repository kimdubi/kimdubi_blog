---
title: "MongoDB Sharded Cluster - Mongos"
date: 2019-12-12T17:53:15+09:00
---

### mongos 란?

mongos 는 mongodb shard cluster 내에서 router 역할을 하는 컴포넌트로 아래와 같은 역할 담당 및 특징이 있음

* 쿼리 라우팅
* 쿼리 결과 merge 후 client에게 return  
* order by 같은 경우에는 샤드 서버 중 프라이머리 샤드를 정한 후  
그 프라이머리 샤드가 다른 샤드로부터 쿼리 결과를 전달받고 order by 수행한 후에 최종 결과를 라우터로 반환함
* 2.x 버전에선 chunk 리밸런싱, 스플릿 등 chunk 관련 역할도 했으나 3.4 버전부터는  
해당 역할이 모두 config 서버에서 담당하게 되어 mongos 는 말그대로 router 역할만 하게됨
* 줄어든 역할로 서버의 리소스를 매우 적게 사용하기 때문에 아래와 같이 mongos 를 다른 컴포넌트들과 같이 구성하기도 함
    * 어플리케이션 서버 n대 + mongos n대
    * config + mongos
    * mongos + shard 서버
    * mongos 만 따로 구성

    * mongodb 에서는 mongos down 시 mongos 를 바라보는 어플리케이션도 어차피 역할을 못하기 때문에 한 세트의 의미로 1번의 구조를 추천하지만 이슈가 발생했을 때 원인 분석, VM의 성능 향상과 관리상의 이유 등으로 mongos 를 따로 구성하기도 함



### 쿼리 라우팅

MongoDB sharded cluster 에서 collection은 shard key 를 기준으로 쪼개져서 여러 샤드에 분산되어 저장됨  
이 단위를 chunk 라고 부르며 어느 shard에 어떤 chunk 가 있는지, chunk 의 max key , min key 범위 같은 메타정보를  
config 서버에서 읽어오고 사용자의 쿼리 내 샤드 키에 따라 mongos 는   
특정 샤드에만 쿼리를 전달하는 타겟 쿼리 / 전체 샤드에 쿼리를 전달하는 브로드캐스트 쿼리로 요청을 처리하게 됨

* 테스트 데이터 준비

~~~
mongos> use routing_test;
switched to db routing_test

mongos> sh.enableSharding('routing_test');
{
"ok" : 1,
"operationTime" : Timestamp(1567941332, 3),
"$clusterTime" : {
"clusterTime" : Timestamp(1567941332, 3),
"signature" : {
"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
"keyId" : NumberLong(0)
}
}
}

mongos> sh.shardCollection('routing_test.employee',{id:1});
{
"collectionsharded" : "routing_test.employee",
"collectionUUID" : UUID("16ee8e2c-3b16-4ca2-879f-159cc45af712"),
"ok" : 1,
"operationTime" : Timestamp(1567941591, 3),
"$clusterTime" : {
"clusterTime" : Timestamp(1567941591, 3),
"signature" : {
"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
"keyId" : NumberLong(0)
}
}
}
~~~

=> routing_test database & employee collection 생성 후 샤딩 설정



~~~
mongos> for (var i=0; i<200000; i++){
... db.employee.save( {id:i, name:"kim"+i, dept: "DB"+i })};
WriteResult({ "nInserted" : 1 })
~~~

=> 테스트 데이터 생성

~~~
mongos> sh.splitAt("routing_test.employee",{id:50000})
mongos> sh.splitAt("routing_test.employee",{id:100000})
mongos> sh.splitAt("routing_test.employee",{id:150000})
~~~

=> sh.shardCollection('routing_test.employee',{id:1});  
range sharding 으로 collection 생성 했기 때문에 chunk 를 범위 지정하여 split 수행  

~~~
mongos> use config;
switched to db config

mongos> db.chunks.find()

{ "_id" : "routing_test.employee-id_MinKey", "lastmod" : Timestamp(2, 0), "lastmodEpoch" : ObjectId("5d74e3d606d6020a9ea17680"), "ns" : "routing_test.employee", "min" : { "id" : { "$minKey" : 1 } }, "max" : { "id" : 50000 }, "shard" : "repl_shard1", "history" : [ { "validAfter" : Timestamp(1567943966, 26236), "shard" : "repl_shard1" } ] }

{ "_id" : "routing_test.employee-id_50000.0", "lastmod" : Timestamp(3, 0), "lastmodEpoch" : ObjectId("5d74e3d606d6020a9ea17680"), "ns" : "routing_test.employee", "min" : { "id" : 50000 }, "max" : { "id" : 100000 }, "shard" : "repl_shard1", "history" : [ { "validAfter" : Timestamp(1567943973, 6739), "shard" : "repl_shard1" } ] }

{ "_id" : "routing_test.employee-id_100000.0", "lastmod" : Timestamp(3, 1), "lastmodEpoch" : ObjectId("5d74e3d606d6020a9ea17680"), "ns" : "routing_test.employee", "min" : { "id" : 100000 }, "max" : { "id" : 150000 }, "shard" : "repl_shard2", "history" : [ { "validAfter" : Timestamp(1567941590, 6), "shard" : "repl_shard2" } ] }

{ "_id" : "routing_test.employee-id_150000.0", "lastmod" : Timestamp(2, 3), "lastmodEpoch" : ObjectId("5d74e3d606d6020a9ea17680"), "ns" : "routing_test.employee", "min" : { "id" : 150000 }, "max" : { "id" : { "$maxKey" : 1 } }, "shard" : "repl_shard2", "history" : [ { "validAfter" : Timestamp(1567941590, 6), "shard" : "repl_shard2" } ] }
~~~
 
* routing_test.employee-id_MinKey
* routing_test.employee-id_50000.0
* routing_test.employee-id_100000.0 
* routing_test.employee-id_150000.0

총 4개의 chunk 가 생성되었으며 각각 min / max 값에 포함되는 데이터들이 해당 chunk 에 들어가게됨





*  Target Query

~~~
mongos> db.employee.find({id:1}).explain()
{
"queryPlanner" : {
"mongosPlannerVersion" : 1,
"winningPlan" : {
"stage" : "SINGLE_SHARD",
"shards" : [
{
"shardName" : "repl_shard1",
"connectionString" : "repl_shard1/mongo_shard1:27018,mongo_shard2:27018,mongo_shard3:27018",
"serverInfo" : {
"host" : "7cd58cda5dcd",
"port" : 27018,
"version" : "4.1.13",
"gitVersion" : "441714bc4c70699950f3ac51a5cac41dcd413eaa"
},
.
.
.
~~~

=> query 에 shard key 인 id 값이 포함되어 있으며 해당 chunk 를 갖고 있는 shard 에만 쿼리를 전송한 것 확인


* Broadcast Query

~~~
mongos> db.employee.find({name:"kim1"}).explain()

{
"queryPlanner" : {
"mongosPlannerVersion" : 1,
"winningPlan" : {
"stage" : "SHARD_MERGE",
"shards" : [
{
"shardName" : "repl_shard1",
"connectionString" : "repl_shard1/mongo_shard1:27018,mongo_shard2:27018,mongo_shard3:27018",
"serverInfo" : {
"host" : "7cd58cda5dcd",
"port" : 27018,
"version" : "4.1.13",
"gitVersion" : "441714bc4c70699950f3ac51a5cac41dcd413eaa"
},
.
.
{
"shardName" : "repl_shard2",
"connectionString" : "repl_shard2/mongo_shard4:27018,mongo_shard5:27018,mongo_shard6:27018",
"serverInfo" : {
"host" : "d5904a7bf11d",
"port" : 27018,
"version" : "4.1.13",
"gitVersion" : "441714bc4c70699950f3ac51a5cac41dcd413eaa"
},
~~~

=> query 에 shard key 가 없는 경우 전체 샤드로 쿼리 전송하는 것 확인


