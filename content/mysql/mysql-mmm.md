---
title: "Mysql MMM"
date: 2019-12-29T15:56:38+09:00
---
## MySQL MMM ? ##

* Multi-Master Replication Manager
* perl 기반의 Auto Failover 지원하는 open source 
* MMM Monitor에서 DB서버 health check와 Failover를 수행함
* Monitor <-> Agent (db node)

### 구성 ##
* db1 (master)
* db2 (master)
* db3 (slave)
* mmm monitor 1


## setting 

### user 생성

~~~
create user 'mmm_monitor'@'172.17.0.%' identified by 'qhdks123';
GRANT REPLICATION CLIENT ON *.* TO 'mmm_monitor'@'172.17.0.%' ;

create user 'mmm_agent'@'172.17.0.%' identified by 'qhdks123';
GRANT SUPER, REPLICATION CLIENT, PROCESS ON *.* TO 'mmm_agent'@'172.17.0.%';

create user 'replUser'@'172.17.0.%' identified by 'qhdks123';
GRANT REPLICATION SLAVE ON *.* TO 'replUser'@'172.17.0.%';

flush privileges;
~~~
* mmm_monitor : 각 서버의 replication 상태를 체크하기 위한 계정
* mmm_agent : 서버의 이슈가 발생했을 때 페일오버 등의 통제 작업을 수행하는 계정
* replUser : DB 간 replication용 계정


### config 설정 (각각)
~~~
server-id           = 1
log_bin             = /var/log/mysql/mysql-bin.log 
log_bin_index       = /var/log/mysql/mysql-bin.log.index 
relay_log           = /var/log/mysql/mysql-relay-bin 
relay_log_index     = /var/log/mysql/mysql-relay-bin.index 
expire_logs_days    = 10 
max_binlog_size     = 100M 
log_slave_updates   = 1
bind-address = 0.0.0.0
auto_increment_increment = 2
~~~
=> replication 을 위해 server-id 는 모든 서버 각각 다르게 설정  
multi-master 구성에서 auto-increment 값이 겹쳐서 출돌나면 안되기 때문에 
master 끼리는 auto-increment 증가값을 다르게 설정함

### db2 , db3 에서 수행
~~~
CHANGE MASTER TO master_host='172.17.0.2', 
master_port=3306, 
master_user='replUser', 
master_password='qhdks123', 
master_log_file='mysql-bin.000015', 
master_log_pos=2196;
~~~

### db1 에서 수행
~~~
CHANGE MASTER TO master_host='172.17.0.3', 
master_port=3306, 
master_user='replUser', 
master_password='qhdks123', 
master_log_file='mysql-bin.000013', 
master_log_pos=360;
~~~
=> multi master 구성이기 때문에 db1도 db2를 바라보게 설정  
db1 <-> db2 양방향 replication 구성 

### MMM 설치 ###

* mysql-monitor
~~~
yum -y install epel-release
yum -y install mysql-mmm mysql-monitor
~~~

* mysql 각 노드
~~~
yum -y install epel-release
yum -y install mysql-mmm mysql-mmm-agent
~~~

* conf 파일 설정

~~~
[root@f9c137e112da mysql-mmm]# pwd
/etc/mysql-mmm
[root@f9c137e112da mysql-mmm]# ls -ltr
total 8
-rw-r----- 1 root root 230 May  4  2018 mmm_agent.conf
-rw-r----- 1 root root 766 Dec 29 04:17 mmm_common.conf
~~~

* vi mmm_common.conf (모니터링 호스트 포함 전부)
~~~
active_master_role      writer

<host default>
    cluster_interface       eth0
    pid_path                /run/mysql-mmm-agent.pid
    bin_path                /usr/libexec/mysql-mmm/
    replication_user        replUser
    replication_password    qhdks123
    agent_user              mmm_agent
    agent_password          qhdks123
</host>

<host db1>
    ip      172.17.0.2
    mode    master
    peer    db2
</host>

<host db2>
    ip      172.17.0.3
    mode    master
    peer    db1
</host>

<host db3>
    ip      172.17.0.4
    mode    slave
</host>

<role writer>
    hosts   db1, db2
    ips     172.17.0.100  ## write 용 VIP
    mode    exclusive
</role>

<role reader>
    hosts   db3
    ips     172.17.0.103   ## read 용 VIP
    mode    balanced
</role>
~~~


* vi mmm_agent.conf (db node)
~~~
include mmm_common.conf

this db2  ## 위 설정대로 해당 호스트에 맞게 설정 (db1,db2,db3 ..)
~~~

* vi mmm_mon.conf (monitoring node)

~~~
include mmm_common.conf

<monitor>
    ip                  127.0.0.1
    pid_path            /run/mysql-mmm-monitor.pid
    bin_path            /usr/libexec/mysql-mmm
    status_path         /var/lib/mysql-mmm/mmm_mond.status
    ping_ips            172.17.0.2,172.17.0.3,172.17.0.4   ### db node ip
    auto_set_online     60

    # kill_host_bin     /usr/libexec/mysql-mmm/monitor/kill_host
</monitor>

<host default>
    monitor_user        mmm_monitor
    monitor_password    qhdks123
</host>

debug 0
~~~


### MMM 기동 ###

* agent (db node)
~~~
[root@f9c137e112da mysql-mmm]# mmm_agentd start
[root@f9c137e112da mysql-mmm]# ps -ef | grep mmm
root       363     0 24 04:54 ?        00:00:00 mmm_agentd
root       364   363  0 04:54 ?        00:00:00 mmm_agentd
~~~

* monitor (monitoring node)

~~~
[root@511c2ddb54da sbin]# mmm_mond start
[root@511c2ddb54da sbin]# mmm_control show
ERROR: Can't connect to monitor daemon!
[root@511c2ddb54da sbin]# ps -ef | grep mmm
root     12373     0 12 04:55 ?        00:00:00 mmm_mond
root     12374 12373  2 04:55 ?        00:00:00 mmm_mond
root     12385 12374  2 04:55 ?        00:00:00 perl /usr/libexec/mysql-mmm/monitor/checker ping_ip
root     12388 12374  2 04:55 ?        00:00:00 perl /usr/libexec/mysql-mmm/monitor/checker mysql
root     12390 12374  2 04:55 ?        00:00:00 perl /usr/libexec/mysql-mmm/monitor/checker ping
root     12392 12374  2 04:55 ?        00:00:00 perl /usr/libexec/mysql-mmm/monitor/checker rep_backlog
root     12395 12374  2 04:55 ?        00:00:00 perl /usr/libexec/mysql-mmm/monitor/checker rep_threads
root     12399 12355  0 04:55 pts/2    00:00:00 grep --color=auto mmm
~~~

~~~
[root@511c2ddb54da mysql-mmm]# mmm_control show
  db1(172.17.0.2) master/HARD_OFFLINE. Roles:
  db2(172.17.0.3) master/HARD_OFFLINE. Roles:
  db3(172.17.0.4) slave/HARD_OFFLINE. Roles:
~~~
=> db가 offline으로 정상 인식이 안되는 상태

~~~
[root@511c2ddb54da mysql-mmm]# mmm_control checks all
db2  ping         [last change: 2019/12/29 05:03:38]  OK
db2  mysql        [last change: 2019/12/29 05:03:38]  ERROR: Connect error (host = 172.17.0.3:3306, user = mmm_monitor)! Authentication plugin 'caching_sha2_password' cannot be loaded: /usr/lib64/mysql/plugin/caching_sha2_password.so: cannot open shared object file: No such file or directory
db2  rep_threads  [last change: 2019/12/29 05:03:38]  UNKNOWN: Connect error (host = 172.17.0.3:3306, user = mmm_monitor)! Authentication plugin 'caching_sha2_password' cannot be loaded: /usr/lib64/mysql/plugin/caching_sha2_password.so: cannot open shared object file: No such file or directory
db2  rep_backlog  [last change: 2019/12/29 05:03:38]  UNKNOWN: Connect error (host = 172.17.0.3:3306, user = mmm_monitor)! Authentication plugin 'caching_sha2_password' cannot be loaded: /usr/lib64/mysql/plugin/caching_sha2_password.so: cannot open shared object file: No such file or directory
db3  ping         [last change: 2019/12/29 05:03:38]  OK
db3  mysql        [last change: 2019/12/29 05:03:38]  ERROR: Connect error (host = 172.17.0.4:3306, user = mmm_monitor)! Authentication plugin 'caching_sha2_password' cannot be loaded: /usr/lib64/mysql/plugin/caching_sha2_password.so: cannot open shared object file: No such file or directory
db3  rep_threads  [last change: 2019/12/29 05:03:38]  UNKNOWN: Connect error (host = 172.17.0.4:3306, user = mmm_monitor)! Authentication plugin 'caching_sha2_password' cannot be loaded: /usr/lib64/mysql/plugin/caching_sha2_password.so: cannot open shared object file: No such file or directory
db3  rep_backlog  [last change: 2019/12/29 05:03:38]  UNKNOWN: Connect error (host = 172.17.0.4:3306, user = mmm_monitor)! Authentication plugin 'caching_sha2_password' cannot be loaded: /usr/lib64/mysql/plugin/caching_sha2_password.so: cannot open shared object file: No such file or directory
db1  ping         [last change: 2019/12/29 05:03:38]  OK
db1  mysql        [last change: 2019/12/29 05:03:38]  ERROR: Connect error (host = 172.17.0.2:3306, user = mmm_monitor)! Authentication plugin 'caching_sha2_password' cannot be loaded: /usr/lib64/mysql/plugin/caching_sha2_password.so: cannot open shared object file: No such file or directory
db1  rep_threads  [last change: 2019/12/29 05:03:38]  UNKNOWN: Connect error (host = 172.17.0.2:3306, user = mmm_monitor)! Authentication plugin 'caching_sha2_password' cannot be loaded: /usr/lib64/mysql/plugin/caching_sha2_password.so: cannot open shared object file: No such file or directory
db1  rep_backlog  [last change: 2019/12/29 05:03:38]  UNKNOWN: Connect error (host = 172.17.0.2:3306, user = mmm_monitor)! Authentication plugin 'caching_sha2_password' cannot be loaded: /usr/lib64/mysql/plugin/caching_sha2_password.so: cannot open shared object file: No such file or directory
~~~
=> mmm_control checks all 커맨드로 확인

~~~
[root@511c2ddb54da mysql-mmm]# mmm_control checks all
db2  ping         [last change: 2019/12/29 05:22:08]  OK
db2  mysql        [last change: 2019/12/29 05:23:22]  OK
db2  rep_threads  [last change: 2019/12/29 05:23:22]  OK
db2  rep_backlog  [last change: 2019/12/29 05:23:22]  OK: Backlog is null
db3  ping         [last change: 2019/12/29 05:22:08]  OK
db3  mysql        [last change: 2019/12/29 05:23:22]  OK
db3  rep_threads  [last change: 2019/12/29 05:23:22]  OK
db3  rep_backlog  [last change: 2019/12/29 05:23:22]  OK: Backlog is null
db1  ping         [last change: 2019/12/29 05:22:08]  OK
db1  mysql        [last change: 2019/12/29 05:23:22]  OK
db1  rep_threads  [last change: 2019/12/29 05:23:22]  OK
db1  rep_backlog  [last change: 2019/12/29 05:23:22]  OK: Backlog is null
~~~
=> 조치 후 check 

~~~
[root@511c2ddb54da mysql-mmm]# mmm_control show
  db1(172.17.0.2) master/AWAITING_RECOVERY. Roles:
  db2(172.17.0.3) master/AWAITING_RECOVERY. Roles:
  db3(172.17.0.4) slave/AWAITING_RECOVERY. Roles:
~~~
=> recovery 기다리는 상태로 서비스 투입 안된 상태


~~~
[root@511c2ddb54da mysql-mmm]# tail -f mmm_mond.log
2019/12/29 05:22:11 FATAL Can't reach agent on host 'db1'
2019/12/29 05:22:19 FATAL Agent on host 'db2' is reachable again
2019/12/29 05:22:19 FATAL Agent on host 'db3' is reachable again
2019/12/29 05:22:19 FATAL Agent on host 'db1' is reachable again
2019/12/29 05:23:22 FATAL State of host 'db2' changed from HARD_OFFLINE to AWAITING_RECOVERY
2019/12/29 05:23:22 FATAL State of host 'db3' changed from HARD_OFFLINE to AWAITING_RECOVERY
2019/12/29 05:23:22 FATAL State of host 'db1' changed from HARD_OFFLINE to AWAITING_RECOVERY
2019/12/29 05:24:23 FATAL State of host 'db2' changed from AWAITING_RECOVERY to ONLINE because of auto_set_online(60 seconds). It was in state AWAITING_RECOVERY for 61 seconds
2019/12/29 05:24:23 FATAL State of host 'db3' changed from AWAITING_RECOVERY to ONLINE because of auto_set_online(60 seconds). It was in state AWAITING_RECOVERY for 61 seconds
2019/12/29 05:24:23 FATAL State of host 'db1' changed from AWAITING_RECOVERY to ONLINE because of auto_set_online(60 seconds). It was in state AWAITING_RECOVERY for 61 seconds
~~~


~~~
[root@511c2ddb54da mysql-mmm]# mmm_control show
  db1(172.17.0.2) master/ONLINE. Roles: writer(172.17.0.100)
  db2(172.17.0.3) master/ONLINE. Roles:
  db3(172.17.0.4) slave/ONLINE. Roles: reader(172.17.0.103)
~~~ 

=> awiting_recovery 상태에서 mmm_mon.conf에 정의된 auto_set_online 60 설정에 따라 60 초 후 자동 ONLINE 상태로 변경됨  
명시적으로는 아래 커맨드로 수동 변경 가능  
~~~
[root@511c2ddb54da mysql-mmm]# mmm_control set_online db1
OK: State of 'db1' changed to ONLINE. Now you can wait some time and check its new roles!
~~~


## Failover Test


* db3 -> show slave status
~~~
mysql> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.17.0.2
                  Master_User: replUser
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000018
          Read_Master_Log_Pos: 155
               Relay_Log_File: relay-bin.000002
                Relay_Log_Pos: 322
        Relay_Master_Log_File: mysql-bin.000018
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
~~~

* mmm status
~~~
[root@511c2ddb54da mysql-mmm]# mmm_control show
  db1(172.17.0.2) master/ONLINE. Roles: writer(172.17.0.100)
  db2(172.17.0.3) master/ONLINE. Roles:
  db3(172.17.0.4) slave/ONLINE. Roles: reader(172.17.0.103)
~~~
=> 172.17.0.2 서버가 slave 의 master, writer vip도 172.17.0.2 서버에 할당되어있음

* db1 shutdown
~~~
[root@c2b2e13b86b9 mysql]# kill -9 521
[root@c2b2e13b86b9 mysql]#
[1]+  Killed                  ./bin/mysqld --defaults-file=./my.cnf
~~~

* Failover 진행

~~~
2019/12/29 06:09:25  INFO We have some new roles added or old rules deleted!
2019/12/29 06:09:25  INFO Added:   writer(172.17.0.100)

[root@f9c137e112da mysql]# ps -ef | grep mmm
root      1612     0 34 06:35 ?        00:00:01 mmm_agentd
root      1613  1612  0 06:35 ?        00:00:00 mmm_agentd
root      1618  1613  2 06:35 ?        00:00:00 perl /usr/libexec/mysql-mmm//agent/configure_ip mmm_agent eth0 172.17.0.100
~~~
=>db2 에 writer vip 할당됨

~~~
2019/12/29 06:09:25  INFO Changing active master to 'db2'


mysql> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.17.0.3
                  Master_User: replUser
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000015
          Read_Master_Log_Pos: 155
               Relay_Log_File: relay-bin.000002
                Relay_Log_Pos: 322
        Relay_Master_Log_File: mysql-bin.000015
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
~~~
=> db3가 master를 db2 로 변경함


~~~
2019/12/29 06:09:25 FATAL State of host 'db1' changed from ONLINE to HARD_OFFLINE (ping: OK, mysql: not OK)

[root@511c2ddb54da mysql-mmm]# mmm_control show
  db1(172.17.0.2) master/HARD_OFFLINE. Roles:
  db2(172.17.0.3) master/ONLINE. Roles: writer(172.17.0.100)
  db3(172.17.0.4) slave/ONLINE. Roles: reader(172.17.0.103)
~~~
=> db1 의 mysql ping이 안되는 것 확인 후 상태변경
