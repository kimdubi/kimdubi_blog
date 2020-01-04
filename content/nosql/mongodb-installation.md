---
title: "MongoDB Installation"
date: 2019-12-07T14:58:36+09:00
---
### MongoDB ###
MongoDB는 document-oriented NoSQL 계열 오픈소스 데이터베이스로 아래와 같은 구조로 구성됨   
![mongodb_architecture](https://raw.githubusercontent.com/kimdubi/kimdubi.github.io/master/images/mongo_install_1.jpg)  
(출처: https://beginnersbook.com/2017/09/mapping-relational-databases-to-mongodb/)

#
#
#### Document ####
* Document는 기존 RDBMS 의 record 와 같은 개념으로 MongoDB에서 실제 데이터가 저장되는 방식  
* JSON 형태의 key-value 쌍으로 이루어져 있음  
* value에는 array, string, int 등 다양한 type 이 올 수 있음  

```
    {
    name : "kimdubi",
    age : 29,
    interests : ["Infra","DB","NoSQL"]
    }
```

#### Collection ####

* RDBMS 의 테이블과 같은 개념

#### Database 

* mysql 의 스키마와 같은 개념으로 MongoDB 서버는 여러개의 database로 이루어 질 수 있으며   
database 안에는 여러개의 Collection들이 있음


     
### Installation ###

#### 사용할 디렉토리 생성 ####


```
    /engn001/mongo => mongodb 엔진영역
    /data001/mongo => mongodb data 영역
```
  
#### 계정 profile 수정 ####
```
    export MONGO_PATH=/engn001/mongo/
    export PATH=$PATH:$MONGO_PATH/bin
```
#### mongodb tar 압축해제 및 설치 ####
```
    dori:mongo mac$ pwd
    /engn001/mongo
    dori:mongo mac$ ls
    mongodb-osx-ssl-x86_64-4.0.9.tar
```

#### mongodb conf 파일 생성 ####


vi mongod.conf

```     

systemLog:    ## db log 관련설정

  destination: file

  path: "/engn001/mongo/logs"

  logAppend: true   ## 새로운 로그를 기존 로그파일에 추가하는 방식

  verbosity: 0  ## 로깅 레벨로 0~5. 이슈 발생 시 디버깅 하려면 레벨을 높게 설정하는 게 좋음

storage:
  dbPath:   ## db data가 저장될 위치 지정
    "/data001/mongo"

  journal:   ## transaction log 사용 유무 = mysql log-bin 옵션

    enabled: true

    commitIntervalMs: 100   ## 커밋 인터발

  directoryPerDB: true  ## database 별로 디렉토리 별도로 사용

  wiredTiger:   ## mongodb 에서 사용할 엔진 설정 wiredTiger 외 mmapv1 / inMemory 엔진 이 있음

    engineConfig:

      cacheSizeGB: 2  ## buffer pool 사이즈

      journalCompressor: snappy  ## journal data 압축 방식 ( none / snappy - default / zlib )

      directoryForIndexes: false  ## true 설정 시 storage.dbPath 아래 index / collection 디렉토리에 각각 인덱스와 컬렉션을 저장함

    collectionConfig:

      blockCompressor: snappy

    indexConfig:

      prefixCompression: true

processManagement:

  fork: true  ## mongodb를 데몬형태로 띄움

net:

  bindIp: 127.0.0.1

  port: 27017
```

#### mongodb 기동 ####

```
dori:mongo mac$ vi mongod.cfg
dori:mongo mac$ mongod --config=/engn001/mongo/mongod.cfg
about to fork child process, waiting until server is ready for connections.
forked process: 24596
child process started successfully, parent exiting

dori:mongo mac$
```

#### log 확인 ####

```
    2019-05-11T16:07:52.231+0900 I CONTROL  [main] Automatically disabling TLS 1.0, to force-enable TLS 1.0 specify --sslDisabledProtocols 'none'

2019-05-11T16:07:52.242+0900 I CONTROL  [initandlisten] MongoDB starting : pid=24596 port=27017 dbpath=/data001/mongo 64-bit host=dori.local

    2019-05-11T16:07:52.242+0900 I CONTROL  [initandlisten] db version v4.0.9

    2019-05-11T16:07:52.243+0900 I CONTROL  [initandlisten] git version: fc525e2d9b0e4bceff5c2201457e564362909765

    2019-05-11T16:07:52.243+0900 I CONTROL  [initandlisten] allocator: system

    2019-05-11T16:07:52.244+0900 I CONTROL  [initandlisten] modules: none

    2019-05-11T16:07:52.245+0900 I CONTROL  [initandlisten] build environment:

    2019-05-11T16:07:52.246+0900 I CONTROL  [initandlisten]     distarch: x86_64

    2019-05-11T16:07:52.246+0900 I CONTROL  [initandlisten]     target_arch: x86_64

2019-05-11T16:07:52.247+0900 I CONTROL  [initandlisten] options: { config: "/engn001/mongo/mongod.cfg", net: { bindIp: "127.0.0.1", port: 27017 }, processManagement: { fork: true }, storage: { dbPath: "/data001/mongo", directoryPerDB: true, journal: { commitIntervalMs: 100, enabled: true }, wiredTiger: { collectionConfig: { blockCompressor: "snappy" }, engineConfig: { cacheSizeGB: 2.0, directoryForIndexes: false, journalCompressor: "snappy" }, indexConfig: { prefixCompression: true } } }, systemLog: { destination: "file", logAppend: true, path: "/engn001/mongo/logs", verbosity: 0 } }

    2019-05-11T16:07:52.248+0900 I STORAGE  [initandlisten] Detected data files in /data001/mongo created by the 'wiredTiger' storage engine, so setting the active storage engine to 'wiredTiger'.

    2019-05-11T16:07:52.248+0900 I STORAGE  [initandlisten] wiredtiger_open config: create,cache_size=2048M,session_max=20000,eviction=(threads_min=4,threads_max=4),config_base=false,statistics=(fast),log=(enabled=true,archive=true,path=journal,compressor=snappy),file_manager=(close_idle_time=100000),statistics_log=(wait=0),verbose=(recovery_progress),

    2019-05-11T16:07:52.835+0900 I STORAGE  [initandlisten] WiredTiger message [1557558472:835376][24596:0x1131885c0], txn-recover: Main recovery loop: starting at 1/22784 to 2/256

    2019-05-11T16:07:52.904+0900 I STORAGE  [initandlisten] WiredTiger message [1557558472:904075][24596:0x1131885c0], txn-recover: Recovering log 1 through 2

    2019-05-11T16:07:52.948+0900 I STORAGE  [initandlisten] WiredTiger message [1557558472:948240][24596:0x1131885c0], txn-recover: Recovering log 2 through 2

    2019-05-11T16:07:52.987+0900 I STORAGE  [initandlisten] WiredTiger message [1557558472:987223][24596:0x1131885c0], txn-recover: Set global recovery timestamp: 0

    2019-05-11T16:07:53.038+0900 I RECOVERY [initandlisten] WiredTiger recoveryTimestamp. Ts: Timestamp(0, 0)

    2019-05-11T16:07:53.047+0900 I CONTROL  [initandlisten]

    2019-05-11T16:07:53.047+0900 I CONTROL  [initandlisten] ** WARNING: Access control is not enabled for the database.

    2019-05-11T16:07:53.048+0900 I CONTROL  [initandlisten] **          Read and write access to data and configuration is unrestricted.

    2019-05-11T16:07:53.049+0900 I CONTROL  [initandlisten]

    2019-05-11T16:07:53.058+0900 I FTDC     [initandlisten] Initializing full-time diagnostic data capture with directory '/data001/mongo/diagnostic.data'

    2019-05-11T16:07:53.060+0900 I NETWORK  [initandlisten] waiting for connections on port 27017

    2019-05-11T16:08:01.289+0900 I NETWORK  [listener] connection accepted from 127.0.0.1:53710 #1 (1 connection now open)

    2019-05-11T16:08:01.289+0900 I NETWORK  [conn1] received client metadata from 127.0.0.1:53710 conn1: { application: { name: "MongoDB Shell" }, driver: { name: "MongoDB Internal Client", version: "4.0.9" }, os: { type: "Darwin", name: "Mac OS X", architecture: "x86_64", version: "18.5.0" } }
```


#### 구성 파일 ####

* basedir & utility
```     
dori:mongo mac$ pwd
/engn001/mongo
dori:mongo mac$ ls -ltr
total 459048
-rw-r--r--@  1 mac  staff      16726  4 11 11:01 MPL-2
-rw-r--r--@  1 mac  staff      30608  4 11 11:01 LICENSE-Community.txt
-rw-r--r--@  1 mac  staff      60005  4 11 11:01 THIRD-PARTY-NOTICES
-rw-r--r--@  1 mac  staff       2601  4 11 11:01 README
-rw-r--r--@  1 mac  staff  231720960  5 11 11:00 mongodb-osx-ssl-x86_64-4.0.9.tar
drwxr-xr-x  15 mac  staff        480  5 11 11:36 bin
-rw-r--r--   1 mac  staff       1135  5 11 16:07 mongod.cfg
-rw-------   1 mac  staff       9656  5 11 16:24 logs

dori:mongo mac$ cd bin
dori:bin mac$ ls -ltr
total 452384
-rwxr-xr-x@ 1 mac  staff   9779276  4 11 11:02 bsondump
-rwxr-xr-x@ 1 mac  staff  10281148  4 11 11:02 mongostat
-rwxr-xr-x@ 1 mac  staff  10057740  4 11 11:02 mongofiles
-rwxr-xr-x@ 1 mac  staff  10163612  4 11 11:02 mongoexport
-rwxr-xr-x@ 1 mac  staff  10297356  4 11 11:02 mongoimport
-rwxr-xr-x@ 1 mac  staff  10686316  4 11 11:02 mongorestore
-rwxr-xr-x@ 1 mac  staff  10560476  4 11 11:02 mongodump
-rwxr-xr-x@ 1 mac  staff   9978172  4 11 11:02 mongotop
-rwxr-xr-x@ 1 mac  staff  14068648  4 11 11:02 mongoreplay
-rwxr-xr-x@ 1 mac  staff  39299984  4 11 11:10 mongo
-rwxr-xr-x@ 1 mac  staff  34009932  4 11 11:16 mongos
-rwxr-xr-x@ 1 mac  staff  62399084  4 11 11:25 mongod
-rwxr-xr-x@ 1 mac  staff      7769  4 11 11:28 install_compass
dori:bin mac$
```

* data directory
```     
dori:mongo mac$ pwd
/data001/mongo
dori:mongo mac$ ls -ltr
total 272
-rw-------  1 mac  wheel     21  5 11 16:06 WiredTiger.lock
-rw-------  1 mac  wheel     45  5 11 16:06 WiredTiger
-rw-------  1 mac  wheel    114  5 11 16:06 storage.bson
drwx------  4 mac  wheel    128  5 11 16:06 admin
drwx------  4 mac  wheel    128  5 11 16:06 local
-rw-------  1 mac  wheel   4096  5 11 16:07 WiredTigerLAS.wt
drwx------  5 mac  wheel    160  5 11 16:07 journal
-rw-------  1 mac  wheel  16384  5 11 16:07 _mdb_catalog.wt
-rw-------  1 mac  wheel      6  5 11 16:07 mongod.lock
-rw-------  1 mac  wheel  36864  5 11 16:13 sizeStorer.wt
-rw-------  1 mac  wheel  61440  5 11 16:13 WiredTiger.wt
-rw-------  1 mac  wheel   1065  5 11 16:13 WiredTiger.turtle
drwx------  5 mac  wheel    160  5 11 16:28 config
drwx------  4 mac  wheel    128  5 11 16:28 diagnostic.data
``` 
