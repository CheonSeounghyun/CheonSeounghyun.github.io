---
layout: post
date: 2023-12-11
title: "MongoDB 샤드 클러스터 구성"
published: true
lang: ko
excerpt: MongoDB의 샤드 클러스터 구성을 알아보고 구현해 봅니다
tags: nosql mongodb
author: seounghyun
---

## Sharding(샤딩)
대표적인 NoSQL인 MongoDB는 샤딩을 지원하고 있습니다. 샤딩은 여러 시스템에 데이터를 분할하여 배포하는 방법입니다. MongoDB는 샤딩을 사용하여 매우 큰 데이터 처리와 높은 처리량이 포함된 배포를 지원합니다.

MongoDB는 샤딩을 통한 수평 확장을 지원합니다. 수평 확장이란, 시스템 데이터를 여러 서버로 나누고, 필요에 따라 서버를 추가하여 용량을 늘리는 작업입니다. 단일 시스템의 전체 속도나 용량은 높지 않지만, 각 시스템은 전체 작업 부하의 하위 집합을 처리하므로 단일 고속 대용량 서버보다 더 나은 효율성을 제공할 수 있습니다. 필요에 따라 서버를 추가하기만 하면 되기 때문에 단일 시스템의 고급 하드웨어보다 전체 비용이 낮을 수 있습니다. 대신 인프라 및 유지 관리의 복잡성이 증가할 수 있습니다.

![alt]({{ "/assets/2023-12-11-MongoDB-shard-cluster-const/2023-12-11-MongoDB-shard-cluster-const-01.svg" | absolute_url}}){: .center-image }

그림 1. MongoDB 샤드 클러스터 구조
{: style="text-align: center; font-style: italic;"}

- MongoDB 샤드 클러스터의 구성 요소
  1. shard : 샤딩된 데이터의 하위 집합
  2. mongos : mongos 클라이언트 어플리케이션과 샤딩된 클러스터 간의 인터페이스 제공 역할을 하는 쿼리 라우터
  3. config : 클러스터에 대한 메타데이터 및 구성 설정을 저장

## 샤딩의 장점
### 읽기/쓰기
MongoDB는 샤드 클러스터의 샤드 전체에 읽기 및 쓰기 작업을 분산하여 각 샤드가 클러스터 작업의 하위 집합을 처리할 수 있게 합니다. 더 많은 샤드를 추가하면 읽기 및 쓰기 작업을 클러스터 전체에서 수평으로 확장할 수 있습니다.

### 저장 용량
샤딩은 클러스터의 샤드 전체에 데이터를 분산하여 각 샤드에 전체 클러스터 데이터의 하위 집합을 포함할 수 있게 구성됩니다.

### 고가용성
만약 하나 이상의 샤드 복제본이 사용할 수 없게 되더라도 샤딩된 클러스터는 계속 읽기 및 쓰기를 수행할 수 있습니다. 사용할 수 없는 샤드의 데이터에는 액세스 할 수 없지만, 사용 가능한 샤드에 대한 데이터 읽기 쓰기는 가능합니다.

## OS 사전 환경 구성 및 MongoDB 다운로드

#### 설치환경
- Hyper-V CentOS-7-x86_64_Minimal-2009
- MongoDB 5.0.23

### Transparent Huge Page(THP) 옵션 비활성화
THP는 대용량 메모리를 효율적으로 관리하기 위해 메모리 페이지 사이즈를 증가시키는 방식입니다. 그런데 이방식은 Database를 운용하는 시스템에서는 오히려 성능 저하가 나타나게 되는데, 메모리 접근이 연속적으로 되지 않고 파편화 되기 때문입니다.  
그래서 MongoDB에서는 THP가 적용되어 있으면 경고 로그를 출력하고 비활성화를 권장하고 있습니다. THP를 비활성화 시키는 서비스를 등록하고 시작하겠습니다.  

vi /etc/systemd/system/disable-transparent-huge-pages.service  
위 경로에 아래 내용의 파일을 편집기로 생성합니다.

```
[Unit]
Description=Disable Transparent Huge Pages (THP)
DefaultDependencies=no
After=sysinit.target local-fs.target
Before=mongod.service

[Service]
Type=oneshot
ExecStart=/bin/sh -c "echo 'never' >/sys/kernel/mm/transparent_hugepage/enabled && echo 'never' >/sys/kernel/mm/transparent_hugepage/defrag"

[Install]
WantedBy=basic.target
```
파일 작성 완료시 아래와 같이 서비스를 시작시킵니다.
```
systemctl daemon-reload
systemctl enable disable-transparent-huge-pages
systemctl start disable-transparent-huge-pages
```
THP 기능이 비활성화 됐는지 아래와 같이 확인합니다.
```
cat /sys/kernel/mm/transparent_hugepage/enabled
always madvise [never]


cat /sys/kernel/mm/transparent_hugepage/defrag
always defer defer+madvise madvise [never]
```

### Prequisites OS Package
MongoDB 실행에 필요한 OS 패키지를 설치합니다.
```
yum install libcurl openssl xz-libs
```

### Ulimit 수정
```
vi /etc/security/limits.conf

mongod soft nofile 64000
mongod hard nofile 64000
mongod hard nproc 64000
mongod hard nproc 64000
```

### MongoDB 다운로드
MongoDB는 yum레포지토리 생성을 통한 설치와 tar.gz 압축파일을 사용하는 설치방법이 있습니다. yum repo를 통한 설치가 환경변수 설정이나 관리면에서 수월하기 때문에 yum 방식으로 설치하겠습니다.  
MongoDB는 CentOS에서 기본적으로 제공하는 레포지토리에 존재하지 않아서 yum install 명령어로 설치가 불가능 하기 떄문에 직접 레포지토리를 생성해야 합니다.  
```
vi /etc/yum.repos.d/mongodb-org-5.0.repo

[mongodb-org-5.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/5.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-5.0.asc
```

아래 명렁어로 mongodb를 설치합니다.
```
yum install -y mongodb-org
```

정상적으로 MongoDB가 설치됐는지 확인하겠습니다.
```
mongos --version

mongos version v5.0.23
Build Info: {
    "version": "5.0.23",
    "gitVersion": "3367195a14d0ba2734d2ba2719294fb974ad0834",
    "openSSLVersion": "OpenSSL 1.0.1e-fips 11 Feb 2013",
    "modules": [],
    "allocator": "tcmalloc",
    "environment": {
        "distmod": "rhel70",
        "distarch": "x86_64",
        "target_arch": "x86_64"
    }
}
```
### Config Server 구성
config server를 레플리카셋으로 구성하겠습니다.  
먼저 config server 구성을 위한 데이터 저장 경로를 생성합니다.
```
mkdir -p /usr/local/mongodb/config_server/data_27031
mkdir -p /usr/local/mongodb/config_server/data_27032
mkdir -p /usr/local/mongodb/config_server/data_27033
```

config server를 실행 시킵니다.
```
mongod --configsvr --replSet configrs \
--dbpath /usr/local/mongodb/config_server/data_27031 \
--logpath /usr/local/mongodb/config_server/data_27031/mongo_config.log --logappend \
--fork --port 27031 --bind_ip_all --directoryperdb

mongod --configsvr --replSet configrs \
--dbpath /usr/local/mongodb/config_server/data_27032 \
--logpath /usr/local/mongodb/config_server/data_27032/mongo_config.log --logappend \
--fork --port 27032 --bind_ip_all --directoryperdb

mongod --configsvr --replSet configrs \
--dbpath /usr/local/mongodb/config_server/data_27033 \
--logpath /usr/local/mongodb/config_server/data_27033/mongo_config.log --logappend \
--fork --port 27033 --bind_ip_all --directoryperdb
```

실행된 config server중 하나의 서버에 접속해서 레플리카셋 설정을 합니다.
```
## OS 에서 config server 접속
mongo --host 127.0.0.1 --port 27031

## config server 접속후 아래 명령어 수행
use admin
rs.initiate(
{
    _id : "configrs",
    configsvr : true,
    members : [
        { _id:0, host: "127.0.0.1:27031"},
        { _id:1, host: "127.0.0.1:27032"},
        { _id:2, host: "127.0.0.1:27033"}
        ]
    }
)
```
구성된 config server 레플리카셋에 대한 정보를 확인합니다.
```
rs.hello()

{
    "topologyVersion" : {
            "processId" : ObjectId("656ff946868d692ac7c2a397"),
            "counter" : NumberLong(6)
    },
    "hosts" : [
            "127.0.0.1:27031",
            "127.0.0.1:27032",
            "127.0.0.1:27033"
    ],
    "setName" : "configrs",
    "setVersion" : 1,
    "isWritablePrimary" : true,
    "secondary" : false,
    "primary" : "127.0.0.1:27031",
    "me" : "127.0.0.1:27031",
    "electionId" : ObjectId("7fffffff0000000000000001"),
    "lastWrite" : {
            "opTime" : {
                    "ts" : Timestamp(1702355807, 1),
                    "t" : NumberLong(1)
            },
            "lastWriteDate" : ISODate("2023-12-12T04:36:47Z"),
            "majorityOpTime" : {
                    "ts" : Timestamp(1702355807, 1),
                    "t" : NumberLong(1)
            },
            "majorityWriteDate" : ISODate("2023-12-12T04:36:47Z")
    },
    "configsvr" : 2,
    "maxBsonObjectSize" : 16777216,
    "maxMessageSizeBytes" : 48000000,
    "maxWriteBatchSize" : 100000,
    "localTime" : ISODate("2023-12-12T04:36:48.252Z"),
    "logicalSessionTimeoutMinutes" : 30,
    "connectionId" : 1235,
    "minWireVersion" : 0,
    "maxWireVersion" : 13,
    "readOnly" : false,
    "ok" : 1,
    "$gleStats" : {
            "lastOpTime" : Timestamp(0, 0),
            "electionId" : ObjectId("7fffffff0000000000000001")
    },
    "lastCommittedOpTime" : Timestamp(1702355807, 1),
    "$clusterTime" : {
            "clusterTime" : Timestamp(1702355807, 1),
            "signature" : {
                    "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
                    "keyId" : NumberLong(0)
            }
    },
    "operationTime" : Timestamp(1702355807, 1)
}

```
### Shard Server 구성
샤드 데이터 디렉토리를 생성합니다.
```
mkdir -p /usr/local/mongodb/shard_server/data_27021
mkdir -p /usr/local/mongodb/shard_server/data_27022
mkdir -p /usr/local/mongodb/shard_server/data_27023

mkdir -p /usr/local/mongodb/shard_server/data_27024
mkdir -p /usr/local/mongodb/shard_server/data_27025
mkdir -p /usr/local/mongodb/shard_server/data_27026

mkdir -p /usr/local/mongodb/shard_server/data_27027
mkdir -p /usr/local/mongodb/shard_server/data_27028
mkdir -p /usr/local/mongodb/shard_server/data_27029
```
shard server를 실행 시킵니다.
```
## Shard-1
mongod --shardsvr --replSet replset1 \
--dbpath /usr/local/mongodb/shard_server/data_27021 \
--logpath /usr/local/mongodb/shard_server/data_27021/mongo_shard.log --logappend \
--fork --port 27021 --bind_ip_all --directoryperdb 

mongod --shardsvr --replSet replset1 \
--dbpath /usr/local/mongodb/shard_server/data_27022 \
--logpath /usr/local/mongodb/shard_server/data_27022/mongo_shard.log --logappend \
--fork --port 27022 --bind_ip_all --directoryperdb 

mongod --shardsvr --replSet replset1 \
--dbpath /usr/local/mongodb/shard_server/data_27023 \
--logpath /usr/local/mongodb/shard_server/data_27023/mongo_shard.log --logappend \
--fork --port 27023 --bind_ip_all --directoryperdb 

## Shard-2
mongod --shardsvr --replSet replset2 \
--dbpath /usr/local/mongodb/shard_server/data_27024 \
--logpath /usr/local/mongodb/shard_server/data_27024/mongo_shard.log --logappend \
--fork --port 27024 --bind_ip_all --directoryperdb 

mongod --shardsvr --replSet replset2 \
--dbpath /usr/local/mongodb/shard_server/data_27025 \
--logpath /usr/local/mongodb/shard_server/data_27025/mongo_shard.log --logappend \
--fork --port 27025 --bind_ip_all --directoryperdb 

mongod --shardsvr --replSet replset2 \
--dbpath /usr/local/mongodb/shard_server/data_27026 \
--logpath /usr/local/mongodb/shard_server/data_27026/mongo_shard.log --logappend \
--fork --port 27026 --bind_ip_all --directoryperdb 

## Shard-3
mongod --shardsvr --replSet replset3 \
--dbpath /usr/local/mongodb/shard_server/data_27027 \
--logpath /usr/local/mongodb/shard_server/data_27027/mongo_shard.log --logappend \
--fork --port 27027 --bind_ip_all --directoryperdb 

mongod --shardsvr --replSet replset3 \
--dbpath /usr/local/mongodb/shard_server/data_27028 \
--logpath /usr/local/mongodb/shard_server/data_27028/mongo_shard.log --logappend \
--fork --port 27028 --bind_ip_all --directoryperdb 

mongod --shardsvr --replSet replset3 \
--dbpath /usr/local/mongodb/shard_server/data_27029 \
--logpath /usr/local/mongodb/shard_server/data_27029/mongo_shard.log --logappend \
--fork --port 27029 --bind_ip_all --directoryperdb 
```

각 샤드서버의 레플리카셋을 설정하기 위해 프라이머리 노드로 사용할 서버에 접속합니다.
```
mongo --host 127.0.0.1 --port 27021
```
1개 서버를 arbiter로 변경 합니다.
```
use admin

rs.initiate(
    {
        _id : "replset1",
        members : [
            {_id : 0, host: "127.0.0.1:27021"},
            {_id : 1, host: "127.0.0.1:27022"},
            {_id : 2, host: "127.0.0.1:27023",arbiterOnly:true}
            ]
    }
)
{ "ok" : 1 }
```

위와 같은 방법으로 나머지 2개의 레플리카셋도 변경하겠습니다.
```
$ mongo --host 127.0.0.1 --port 27024

use admin

rs.initiate(
    {
        _id : "replset2",
        members : [
            {_id : 0, host: "127.0.0.1:27024"},
            {_id : 1, host: "127.0.0.1:27025"},
            {_id : 2, host: "127.0.0.1:27026",arbiterOnly:true}
            ]
    }
)
{ "ok" : 1 }
```
```
$ mongo --host 127.0.0.1 --port 27027
use admin
rs.initiate(
    {
        _id : "replset3",
        members : [
            {_id : 0, host: "127.0.0.1:27027"},
            {_id : 1, host: "127.0.0.1:27028"},
            {_id : 2, host: "127.0.0.1:27029",arbiterOnly:true}
            ]
    }
)
{ "ok" : 1 }
```

위의 설정 변경을 완료 후 아래 명령어를 통해서 레플리카셋 정보를 확인합니다.
```
rs.hello()

{
        "topologyVersion" : {
                "processId" : ObjectId("656ffd917dea513a86cf2bb6"),
                "counter" : NumberLong(9)
        },
        "hosts" : [
                "127.0.0.1:27021",
                "127.0.0.1:27022"
        ],
        "arbiters" : [
                "127.0.0.1:27023"
        ],
        "setName" : "replset1",
        "setVersion" : 2,
        "isWritablePrimary" : true,
        ...
```
이제 config server, 레플리카셋 샤드 서버 구성이 완료됐습니다.  

최종적으로 구성된 mongo 디렉토리 구조와 포트 할당 정보는 아래와 같습니다.
```
/usr/local/mongodb
├── config_server
│   ├── data_27031
│   ├── data_27032
│   └── data_27033
├── mongos
│   └── mongos_27041
└── shard_server
    ├── data_27021
    ├── data_27022
    ├── data_27023
    ├── data_27024
    ├── data_27025
    ├── data_27026
    ├── data_27027
    ├── data_27028
    └── data_27029

* config server Port : 27031 / 27032 / 27033
* shard server Port :   replset1 - 27021 / 27022 / 27023(Arbiter)
                        replset2 - 27024 / 27025 / 27026(Arbiter)
                        replset3 - 27027 / 27028 / 27029(Arbiter)
* mongos Port : 27041
```

### MongoS 실행
mongos 라우터 서버를 실행하도록 하겠습니다.  
logpath를 위해 디렉토리를 생성합니다.
```
mkdir -p /usr/local/mongodb/mongos/mongos_27041
```

mongos 를 실행시킵니다.
```
mongos --configdb configrs/127.0.0.1:27031,127.0.0.1:27032,127.0.0.1:27033 \
--logpath /usr/local/mongodb/mongos/mongos_27041/mongos.log --logappend \
--fork --port 27041 --bind_ip_all
```

### 샤드 클러스터 추가
레플리카셋으로 구성한 샤드 서버를 클러스터에 추가하도록 하겠습니다.  
mongos 라우터 서버로 접속합니다.
```
mongo --host 127.0.0.1 --port 27041
```

샤드 클러스터에 서버를 추가합니다. (arbiter 노드를 제외하고 primary 와 secondary노드만 입력하면 됩니다.)
```
sh.addShard("replset1/127.0.0.1:27021,127.0.0.1:27022")
sh.addShard("replset2/127.0.0.1:27024,127.0.0.1:27025")
sh.addShard("replset3/127.0.0.1:27027,127.0.0.1:27028")
```
샤드 클러스터에 추가 완료시에 다음 명령어를 통해서 샤드 클러스터의 상태를 확인할 수 있습니다.
```
sh.status()

--- Sharding Status ---
  sharding version: {
        "_id" : 1,
        "minCompatibleVersion" : 5,
        "currentVersion" : 6,
        "clusterId" : ObjectId("656ffaad868d692ac7c2a42b")
    }
    shards:
            {  "_id" : "replset1",  "host" : "replset1/127.0.0.1:27021,127.0.0.1:27022",  "state" : 1,  "topologyTime" : Timestamp(1701839116, 2) }
            {  "_id" : "replset2",  "host" : "replset2/127.0.0.1:27024,127.0.0.1:27025",  "state" : 1,  "topologyTime" : Timestamp(1701839151, 2) }
            {  "_id" : "replset3",  "host" : "replset3/127.0.0.1:27027,127.0.0.1:27028",  "state" : 1,  "topologyTime" : Timestamp(1701839168, 1) }
    active mongoses:
            "5.0.23" : 1
    ...
```
이제 MongoDB 샤드 클러스터 구성이 완료됐습니다.

## 샤드 활성화 및 샤딩 테스트
샤드를 활성화 시키고 임의의 데이터를 입력해서 정상적으로 샤딩이 되어(분할되어) 저장되는지 테스트 해보겠습니다. 샤드 클러스터 활성화는 샤드로 사용할 데이터베이스에 대해서 한번 실행이 필요합니다. 
```
mongo --host 127.0.0.1 --port 27041
```
라우터 서버에 mongos로 접속합니다.
```
sh.enableSharding("tdb")
```
tdb라는 데이터베이스를 지정하고 샤드로 사용하기 위해 enable 합니다.
```
sh.shardCollection("tdb.employees",{empno: "hashed"})
```
샤딩할 콜렉션을 입력하고 샤드키를 선정합니다.
이제 실제 데이터를 입력해서 샤딩이 되어 데이터가 분할되는지 확인해보겠습니다.(99건 입력)
```
use tdb

for(var a=1; a< 100; a++) db.employees.insert({ empno :a })
```

```
use tdb

db.employees.getShardDistribution()
```
db.collection.getShardDistribution() 메서드를 사용하여 특정 콜렉션의 분할 상태를 확인할 수 있습니다.  
또한 콜렉션의 데이터가 어떤 샤드에 어떻게 분산되었는지 확인할 수 있습니다.  

![alt]({{ "/assets/2023-12-11-MongoDB-shard-cluster-const/2023-12-11-MongoDB-shard-cluster-const-02.png" | absolute_url}}){: .center-image }

그림 2. 샤딩된 데이터 및 샤드 클러스터에 저장된 컬렉션 분할 정보
{: style="text-align: center; font-style: italic;"}

## Reference
https://www.mongodb.com/docs/manual/sharding/  
https://www.mongodb.com/docs/manual/core/sharded-cluster-components/  
https://hoing.io/archives/8381