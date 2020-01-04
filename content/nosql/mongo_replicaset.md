---
title: "MongoDB ReplicaSet"
date: 2019-12-07T19:22:51+09:00
---


mongo db에서도 mysql 처럼 M-S 구조로 replication 을 설정할 수 있습니다.  
그 중에서도 자동 failover를 지원하고 master-slave cluster에서 새 node 를 추가하는 것이   
쉬운  replica set 으로 replication 설정하는 방법을 소개하겠습니다.

![](https://raw.githubusercontent.com/kimdubi/kimdubi.github.io/master/images/mongo_replica.png)     



(출처 : mongodb.com)

### replica set 설정할 DB 준비

```
dori:bin mac$ docker run -d -p 30000:27017 --name mongo-1 mongo mongod --replSet repltest
9cc3084a4779fce90b65ed7fde5b987ef882bda58230fd241168e05dc1f93127

dori:bin mac$ docker run -d -p 30001:27017 --name mongo-2 mongo mongod --replSet repltest
e70f38ab592739ceb33426b4b85af755e59be3ba139683fe599584900c19f360

dori:bin mac$ docker run -d -p 30002:27017 --name mongo-3 mongo mongod --replSet repltest
f3adc7ce039c63abb65166eb1874d5f2e1c19ca533ee63f0079c3caba44b6111

dori:bin mac$ docker run -d -p 30003:27017 --name mongo-4 mongo mongod --replSet repltest
948531ffb1d64a15bb1295329d49864b40352783e7266dab5e5635c695228d4c
```
* --replSet replicaSet이름 옵션으로 repltest 라는 replicaSet db 4대 기동


### replica set 설정
```
dori:bin mac$ mongo --port 30000
MongoDB shell version v4.0.9

>
> repl_conf={
... _id: "repltest",
... members: [{_id: 0, host: "DB 노드 IP :30000"}]
... }

{
"_id" : "repltest",
"members" : [
{
"_id" : 0,
"host" : "DB 노드 IP:30000"
}
]
}

> rs.initiate(repl_conf)
{
"ok" : 1,
"operationTime" : Timestamp(1558191512, 1),
"$clusterTime" : {
"clusterTime" : Timestamp(1558191512, 1),
"signature" : {
"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
"keyId" : NumberLong(0)
}
}
}
```
* replica set 의 멤버로 우선 노드 한개만 설정 후 rs.add() 커맨드로 추가할 것임
* replica set 설정 후 rs.initate() 커맨드 실행


### rs.add() 로 노드 추가

```
repltest:PRIMARY> rs.add("DB 노드 IP:30001")
{
"ok" : 1,
"operationTime" : Timestamp(1558191605, 1),
"$clusterTime" : {
"clusterTime" : Timestamp(1558191605, 1),
"signature" : {
"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
"keyId" : NumberLong(0)
}
}
}
```
* replica set 의 멤버에 rs.add() 로 노드 추가


### rs.status() 로 replica set 현재 상태 확인
```
repltest:PRIMARY> rs.status()

{
"set" : "repltest",
"date" : ISODate("2019-05-18T15:02:50.744Z"),
"myState" : 1,
"term" : NumberLong(1),
"syncingTo" : "",
"syncSourceHost" : "",
"syncSourceId" : -1,
"heartbeatIntervalMillis" : NumberLong(2000),
"optimes" : {
"lastCommittedOpTime" : {
"ts" : Timestamp(1558191767, 1),
"t" : NumberLong(1)
},
"readConcernMajorityOpTime" : {
"ts" : Timestamp(1558191767, 1),
"t" : NumberLong(1)
},
"appliedOpTime" : {
"ts" : Timestamp(1558191767, 1),
"t" : NumberLong(1)
},
"durableOpTime" : {
"ts" : Timestamp(1558191767, 1),
"t" : NumberLong(1)
}
},
"lastStableCheckpointTimestamp" : Timestamp(1558191750, 1),

"members" : [
{
"_id" : 0,
"name" : "DB 노드 IP:30000",
"health" : 1,
"state" : 1,
"stateStr" : "PRIMARY",
"uptime" : 610,
"optime" : {
"ts" : Timestamp(1558191767, 1),
"t" : NumberLong(1)
},
"optimeDate" : ISODate("2019-05-18T15:02:47Z"),
"syncingTo" : "",
"syncSourceHost" : "",
"syncSourceId" : -1,
"infoMessage" : "",
"electionTime" : Timestamp(1558191512, 2),
"electionDate" : ISODate("2019-05-18T14:58:32Z"),
"configVersion" : 6,
"self" : true,
"lastHeartbeatMessage" : ""
},

{
"_id" : 1,
"name" : "DB 노드 IP:30001",
"health" : 1,
"state" : 2,
"stateStr" : "SECONDARY",
"uptime" : 164,
"optime" : {
"ts" : Timestamp(1558191767, 1),
"t" : NumberLong(1)
},
"optimeDurable" : {
"ts" : Timestamp(1558191767, 1),
"t" : NumberLong(1)
},
"optimeDate" : ISODate("2019-05-18T15:02:47Z"),
"optimeDurableDate" : ISODate("2019-05-18T15:02:47Z"),
"lastHeartbeat" : ISODate("2019-05-18T15:02:49.276Z"),
"lastHeartbeatRecv" : ISODate("2019-05-18T15:02:49.344Z"),
"pingMs" : NumberLong(7),
"lastHeartbeatMessage" : "",
"syncingTo" : "DB 노드 IP:30003",
"syncSourceHost" : "DB 노드 IP:30003",
"syncSourceId" : 3,
"infoMessage" : "",
"configVersion" : 6
},

{
"_id" : 2,
"name" : "DB 노드 IP:30002",
"health" : 1,
"state" : 2,
"stateStr" : "SECONDARY",
"uptime" : 19,
"optime" : {

"ts" : Timestamp(1558191767, 1),

"t" : NumberLong(1)
},
"optimeDurable" : {
"ts" : Timestamp(158191767, 1),
"t" : NumberLong(1)
},

.

.

.

.
```
* Primary node 와 Secondary node 의 상태와 설정이 조회됨

     

### Slave 에서 읽기 에러가 나면?
```
repltest:SECONDARY> show dbs
2019-05-19T00:35:27.090+0900 E QUERY    [js] Error: listDatabases failed:{
"operationTime" : Timestamp(1558193724, 1),
"ok" : 0,
"errmsg" : "not master and slaveOk=false",
"code" : 13435,
"codeName" : "NotMasterNoSlaveOk",
"$clusterTime" : {
"clusterTime" : Timestamp(1558193724, 1),
"signature" : {
"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
"keyId" : NumberLong(0)
}
}
} :

_getErrorWithCode@src/mongo/shell/utils.js:25:13
Mongo.prototype.getDBs@src/mongo/shell/mongo.js:139:1
shellHelper.show@src/mongo/shell/utils.js:882:13
shellHelper@src/mongo/shell/utils.js:766:15
@(shellhelp2):1:1
```
* Secondary node 는 기본적으로 쿼리 수행이 잠겨있어 아래 커맨드로 풀어줘야함

```
repltest:SECONDARY> rs.slaveOk()
```
