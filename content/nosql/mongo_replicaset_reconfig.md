---
title: "MongoDB ReplicaSet reconfig"
date: 2019-12-12T18:19:27+09:00
---

이번 글에서는 mongodb replica Set (이하 RS) 의 Primary 선출 방식과
RS 멤버가 이상할 때 해당 멤버를 제외하고 재설정 하는 방법에 대해서 알아보겠습니다.


* RS member 간 hearbeat 실패 감지

~~~  
 I  REPL_HB  [replexec-11] Heartbeat to mongo_shard1:27018 failed after 2 retries, response status: InvalidReplicaSetConfig: Our replica set con  figuration is invalid or does not include us
 I  REPL [replexec-11] Member mongo_shard1:27018 is now in state RS_DOWN - Our replica set configuration is invalid or does not include us
 I  REPL_HB  [replexec-11] Heartbeat to 85604da3dcb0:27018 failed after 2 retries, response status: InvalidReplicaSetConfig: Our replica set con  figuration is invalid or does not include us
 I  REPL [replexec-11] Member 85604da3dcb0:27018 is now in state RS_DOWN - Our replica set configuration is invalid or does not include us

 I  REPL [replexec-15] can't see a majority of the set, relinquishing primary
 I  REPL [replexec-15] Stepping down from primary in response to heartbeat
 I  REPL [replexec-18] can't see a majority of the set, relinquishing primary
 I  REPL [RstlKillOpThread] Starting to kill user operations
 I  REPL [RstlKillOpThread] Stopped killing user operations
 I  REPL [replexec-15] Stepping down from primary, stats: { userOpsKilled: 0, userOpsRunning: 0 }
 I  REPL [replexec-15] transition to SECONDARY from PRIMARY
~~~
=> repliSet member 중 85604da3dcb0:27018, mongo_shard1 로 heart beat 통신이 실패함  
기존 PRIMARY 였던 mongo_shard1 down 되면서 새로 PRIMARY 선출하기 위해 ELECTION 을 진행하나 아래와 같은 로그와 함께 ELECTION 실패

* PRIMARY 선출을 위한 ELECTION 실패
~~~
I  ELECTION [replexec-0] Not starting an election, since we are not electable due to: Not standing for election because I cannot see a majority   (mask 0x1)
~~~
=> MongoDB에선 ELECTION 을 위해 총 투표권 개수의 과반수가 넘는 NODE들의 투표가 필요한데 
현 상황에서는 투표권을 갖고 있는 4개 노드 중 2개 노드가 다운되어 투표 불가능  
과반수인 3 노드 이상이 투표가 가능한 상태여야함

 

* RS 상태 체크

~~~
repl_shard1:SECONDARY> rs.status()
{
"members" : [
{
"_id" : 0,
"name" : "85604da3dcb0:27018",
"stateStr" : "(not reachable/healthy)",
"lastHeartbeatMessage" : "Our replica set configuration is invalid or does not include us",
},
{
"_id" : 1,
"name" : "mongo_shard2:27018",
"stateStr" : "SECONDARY",
},

{
"_id" : 2,
"name" : "mongo_shard3:27018",
"stateStr" : "SECONDARY",
},

{
"_id" : 3,
"name" : "mongo_shard1:27018",
"stateStr" : "(not reachable/healthy)",
"lastHeartbeatMessage" : "Our replica set configuration is invalid or does not include us",
}
],}
~~~
=> mongo_shard1 / 85604da3dcb0 의 heartbeat 상태가  
"Our replica set configuration is invalid or does not include us" 이며 heartbeat 체크가 안됨
PRIMARY 또한 없는 상태로 repliSet member 재설정이 필요함

 


* RS reconfig

rs.remove() 라는 replica set에서 특정 멤버 제거하는 커맨드도 제공되지만 PRIMARY 에서만 실행가능함  
현재와 같이 PRIMARY 가 없는 상황에서는 rs.reconfig() 방법을 통해 재설정하게 됨

ex)
~~~
repl_shard1:SECONDARY> rs.remove("mongo_shard1:27018", {force:true});
{
"ok" : 0,
"errmsg" : "replSetReconfig should only be run on PRIMARY, but my state is SECONDARY; use the \"force\" argument to override",
"code" : 10107,
"codeName" : "NotMaster"
}
~~~
=> force 옵션으로 실행해도 실패

~~~
repl_shard1:SECONDARY> config=rs.conf()
{
"_id" : "repl_shard1",
"members" : [
{
"_id" : 0,
"host" : "85604da3dcb0:27018",
"arbiterOnly" : false,
"buildIndexes" : true,
"hidden" : false,
"priority" : 1,
"votes" : 1
},

{
"_id" : 1,
"host" : "mongo_shard2:27018",
"arbiterOnly" : false,
"buildIndexes" : true,
"hidden" : false,
"priority" : 1,
"votes" : 1
},

{
"_id" : 2,
"host" : "mongo_shard3:27018",
"arbiterOnly" : false,
"buildIndexes" : true,
"hidden" : false,
"priority" : 1,
"votes" : 1
},

{
"_id" : 3,
"host" : "mongo_shard1:27018",
"arbiterOnly" : false,
"buildIndexes" : true,
"hidden" : false,
"priority" : 1,
"votes" : 1
}],
"replicaSetId" : ObjectId("5d881a1aa12189b6f295164d")}}
~~~
=> config 변수 선언 후 rs.config() 내용 저장

~~~
repl_shard1:SECONDARY> config.members=[config.members[1],config.members[2]]
~~~
=> heartbeat 실패한 노드 제외하여 members 재설정 

 
~~~
repl_shard1:SECONDARY> rs.reconfig(config)
{
"errmsg" : "replSetReconfig should only be run on PRIMARY, but my state is SECONDARY; use the \"force\" argument to override",
"code" : 10107,
"codeName" : "NotMaster",
}
~~~
=> secondary 에서 작업시 force 옵션 필요하다는 에러

 
~~~ 
repl_shard1:SECONDARY> rs.reconfig(config , {force:true})
{
"ok" : 1,
"$clusterTime" : {
"clusterTime" : Timestamp(1569202449, 1),
"signature" : {
"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
"keyId" : NumberLong(0)
}
},
"operationTime" : Timestamp(1569202449, 1)
}
~~~
=> force 옵션과 재수행
 
~~~
repl_shard1:SECONDARY> rs.status()
{
"set" : "repl_shard1",
"members" : [
{
"_id" : 1,
"name" : "mongo_shard2:27018",
"stateStr" : "SECONDARY",
},

{
"_id" : 2,
"name" : "mongo_shard3:27018",
"stateStr" : "SECONDARY",
}],}
~~~
=> RS 재설정 및 정상 확인

 
* RS 재설정 후 PRIMARY 선출 정상 확인

 
~~~
I  REPL [conn36] replSetReconfig admin command received from client; new config: { _id: "repl_shard1", version: 5, protocolVersion: 1, writ  eConcernMajorityJournalDefault: true, members: [ { _id: 1, host: "mongo_shard2:27018", arbiterOnly: false, buildIndexes: true, hidden: false, priority: 1.0, tags: {}, slave  Delay: 0, votes: 1 }, { _id: 2, host: "mongo_shard3:27018", arbiterOnly: false, buildIndexes: true, hidden: false, priority: 1.0, tags: {}, slaveDelay: 0, votes: 1 } ], set  tings: { chainingAllowed: true, heartbeatIntervalMillis: 2000, heartbeatTimeoutSecs: 10, electionTimeoutMillis: 10000, catchUpTimeoutMillis: -1, catchUpTakeoverDelayMillis:   30000, getLastErrorModes: {}, getLastErrorDefaults: { w: 1, wtimeout: 0 }, replicaSetId: ObjectId('5d881a1aa12189b6f295164d') } }

I  REPL [conn36] replSetReconfig config object with 2 members parses ok
I  REPL [conn36] New replica set config in use: { _id: "repl_shard1", version: 24459, protocolVersion: 1, writeConcernMajorityJournalDefaul  t: true, members: [ { _id: 1, host: "mongo_shard2:27018", arbiterOnly: false, buildIndexes: true, hidden: false, priority: 1.0, tags: {}, slaveDelay: 0, votes: 1 }, { _id:   2, host: "mongo_shard3:27018", arbiterOnly: false, buildIndexes: true, hidden: false, priority: 1.0, tags: {}, slaveDelay: 0, votes: 1 } ], settings: { chainingAllowed: tru  e, heartbeatIntervalMillis: 2000, heartbeatTimeoutSecs: 10, electionTimeoutMillis: 10000, catchUpTimeoutMillis: -1, catchUpTakeoverDelayMillis: 30000, getLastErrorModes: {}  , getLastErrorDefaults: { w: 1, wtimeout: 0 }, replicaSetId: ObjectId('5d881a1aa12189b6f295164d') } }

I  ELECTION [replexec-35] Starting an election, since we've seen no PRIMARY in the past 10000ms
I  ELECTION [replexec-35] conducting a dry run election to see if we could be elected. current term: 2
I  REPL [replexec-35] Scheduling remote command request for vote request: RemoteCommand 47174 -- target:mongo_shard3:27018 db:admin cmd:{ r  eplSetRequestVotes: 1, setName: "repl_shard1", dryRun: true, term: 2, candidateIndex: 0, configVersion: 24459, lastCommittedOp: { ts: Timestamp(1569202449, 1), t: 2 } }
I  ELECTION [replexec-36] VoteRequester(term 2 dry run) received a yes vote from mongo_shard3:27018; response message: { term: 2, voteGranted:   true, reason: "", ok: 1.0, $clusterTime: { clusterTime: Timestamp(1569202449, 1), signature: { hash: BinData(0, 0000000000000000000000000000000000000000), keyId: 0 } }, ope  rationTime: Timestamp(1569202449, 1) }

I  ELECTION [replexec-36] dry election run succeeded, running for election in term 3

I  REPL [replexec-36] Scheduling remote command request for vote request: RemoteCommand 47175 -- target:mongo_shard3:27018 db:admin cmd:{ r  eplSetRequestVotes: 1, setName: "repl_shard1", dryRun: false, term: 3, candidateIndex: 0, configVersion: 24459, lastCommittedOp: { ts: Timestamp(1569202449, 1), t: 2 } }

I  ELECTION [replexec-32] VoteRequester(term 3) received a yes vote from mongo_shard3:27018; response message: { term: 3, voteGranted: true, re  ason: "", ok: 1.0, $clusterTime: { clusterTime: Timestamp(1569202449, 1), signature: { hash: BinData(0, 0000000000000000000000000000000000000000), keyId: 0 } }, operationTi  me: Timestamp(1569202449, 1) }

I  ELECTION [replexec-32] election succeeded, assuming primary role in term 3
I  REPL [replexec-32] transition to PRIMARY from SECONDARY
~~~
=> 재설정 된 RS 에서는 투표권을 가진 2개의 노드 모두 살아있으므로 PRIMARY 선출가능

 
~~~
I  ELECTION [replexec-35] Starting an election, since we've seen no PRIMARY in the past 10000ms
~~~
=> 기존 PRIMARY 노드를 10초 이내 재기동하면 기존 노드로 PRIMARY 설정하게됨

 
~~~
repl_shard1:PRIMARY> rs.isMaster()
{
"hosts" : [
"mongo_shard2:27018",
"mongo_shard3:27018"
],
"setName" : "repl_shard1",
"ismaster" : true,
"secondary" : false,
"primary" : "mongo_shard2:27018",
"me" : "mongo_shard2:27018",
"electionId" : ObjectId("7fffffff0000000000000007")
~~~
 => election 결과 mongo_shard2 가 PRIMARY로 선출됨

 

#### PRIMARY 가 되기 위한 조건

1. ReplicaSet 에서 과반수 이상이 election 에 참여할 수 있는 상태여야함

2. 해당 노드의 priority 가 0보다 커야함  ( arbiter 제외 )

3. 서버에 저장된 해당 ReplicaSet 의 optime 과 후보 노드의 optime 간 차이가 10초이내여야함

* 서버 내 저장 된 replica set 의 optime
~~~
repl_shard1:PRIMARY> rs.status()

{
"set" : "repl_shard1",
"optimes" : {
"lastCommittedOpTime" : {
"ts" : Timestamp(1569225086, 1),
"t" : NumberLong(7)
},
"lastCommittedWallTime" : ISODate("2019-09-23T07:51:26.608Z"),
"readConcernMajorityOpTime" : {
"ts" : Timestamp(1569225086, 1),
},
"readConcernMajorityWallTime" : ISODate("2019-09-23T07:51:26.608Z"),
"appliedOpTime" : {
"ts" : Timestamp(1569225086, 1),
},
"durableOpTime" : {
"ts" : Timestamp(1569225086, 1),
},
~~~

* 각 노드들의 optime 
~~~
"members" : [
{
"_id" : 1,
"name" : "mongo_shard2:27018",
"stateStr" : "PRIMARY",
"optime" : {
"ts" : Timestamp(1569225086, 1),
},

{
"_id" : 2,
"name" : "mongo_shard3:27018",
"stateStr" : "SECONDARY",
"optime" : {
"ts" : Timestamp(1569225086, 1),
},
"operationTime" : Timestamp(1569225086, 1)
}
~~~


* priority 확인 방법 
~~~
repl_shard1:PRIMARY> rs.config()
{
"_id" : "repl_shard1",
"members" : [
{
"_id" : 1,
"host" : "mongo_shard2:27018",
"priority" : 1,
~~~
=> rs.config() 커맨드로 멤버의 priority 확인 가능하며
priority 설정도 위에서 member 재설정한것과 마찬가지로 rs.reconfig(config) 를 통해 설정가능

ex)
~~~
config=rs.conf()
config.members[0].priority=2
rs.reconfig(config)
~~~
