---
title: Elastic Search 란?
author: SangkiHan
date: 2024-06-27 11:20:00 +0900
categories: [CS]
tags: [CS]
---
------------

## Elastic Search


Elastic Search는 오픈 소스 분산 검색 및 분석 엔진으로, Apache Lucene 라이브러리를 기반으로 구축되었습니다. Elastic Search는 텍스트 검색, 분석, 로그 및 메트릭 데이터의 실시간 분석, 그리고 애플리케이션 성능 모니터링(APM) 등에 주로 사용됩니다. Elastic Search는 대규모의 데이터 세트를 신속하게 검색하고 분석할 수 있는 능력 때문에 다양한 용도로 활용됩니다.

------------
## 주요특징

1. 분산 아키텍처
    + Elastic Search는 분산형으로 설계되어 있으며, 클러스터링 기능을 제공하여 대규모 데이터의 저장 및 검색을 효율적으로 처리할 수 있습니다. 여러 노드로 구성된 클러스터를 통해 데이터 분산 저장 및 처리 능력을 확장할 수 있습니다.

2. 다중 테넌시
    + Elastic Search는 여러 인덱스를 지원하며, 각 인덱스는 독립적인 엔티티로 여러 테넌트의 데이터를 동시에 관리할 수 있습니다.

3. 실시간 검색
    + 실시간 검색 기능을 제공하여 최신 데이터에 대한 빠른 검색 및 조회가 가능합니다. 이는 로그 분석, 모니터링 및 기타 실시간 데이터 처리가 필요한 애플리케이션에서 매우 유용합니다.

4. 강력한 검색 기능
    + Elastic Search는 복잡한 검색 쿼리와 필터링, 정렬 및 집계 기능을 제공합니다. 자연어 처리 기능을 통해 텍스트 데이터의 의미를 이해하고 검색 정확도를 높일 수 있습니다.

5. 스키마리스(schemaless)
    + 데이터 구조를 사전에 정의할 필요 없이 데이터를 인덱스에 삽입할 수 있습니다. 데이터의 스키마는 동적으로 생성되고 관리됩니다.

6. RESTful API
    + Elastic Search는 RESTful API를 통해 쉽게 접근할 수 있으며, JSON 형식의 데이터를 주고받을 수 있습니다. 이를 통해 다양한 프로그래밍 언어와 플랫폼에서 Elastic Search와 통합할 수 있습니다.

7. 확장성
    + Elastic Search는 수평 확장이 가능하여 데이터 양이 증가하더라도 성능을 유지할 수 있습니다. 새로운 노드를 클러스터에 추가하여 용량과 성능을 확장할 수 있습니다.

------------

## 인덱스(Index)
+ 정의: 인덱스는 Elastic Search에서 데이터의 논리적 집합입니다. 데이터베이스의 테이블에 해당한다고 생각할 수 있습니다.
+ 구조: 인덱스는 하나 이상의 문서(document)로 구성됩니다.
+ 역할: 인덱스는 데이터를 저장, 관리, 검색할 수 있는 단위입니다. 각각의 인덱스는 고유한 이름을 가지며, 이 이름을 통해 인덱스에 접근하고 쿼리를 실행할 수 있습니다.

## 문서(Document)
+ 정의: 문서는 Elastic Search에서 데이터의 기본 단위입니다. 데이터베이스의 행(row) 또는 레코드에 해당한다고 볼 수 있습니다.
+ 구조: 문서는 JSON 형식으로 표현되며, 다양한 필드(field)를 가질 수 있습니다. 각 필드는 특정 속성 및 그 값으로 구성됩니다.
+ 역할: 문서는 실제 데이터를 담고 있으며, 인덱스 내에서 검색 가능한 모든 정보를 포함합니다.

### 예시 데이터
``` json
{
  "index": "employees",
  "document": {
    "_id": "1",
    "name": "Alice",
    "position": "Engineer",
    "age": 30
  }
}
{
  "index": "employees",
  "document": {
    "_id": "2",
    "name": "Bob",
    "position": "Manager",
    "age": 40
  }
}

```
------------

## Elastic Search 클러스터 구성요소 

1. 클러스터(Cluster)
    + 정의: 여러 노드로 구성된 Elastic Search의 최상위 단위입니다. 클러스터는 고유한 이름을 가지며, 이 이름을 통해 다른 클러스터와 구별됩니다.
    + 역할: 데이터의 분산 저장, 검색, 복제 및 확장성 제공. 모든 노드는 클러스터에 속하며, 클러스터 내 노드 간 협업을 통해 데이터를 처리합니다.
2. 노드(Node)
    + 정의: 클러스터 내에서 실행되는 Elastic Search 인스턴스입니다. 클러스터는 하나 이상의 노드로 구성될 수 있습니다.
    + 역할: 데이터를 저장하고, 검색 요청을 처리하며, 클러스터 내 다른 노드와 통신. 노드는 역할에 따라 다양한 유형이 있습니다.
3. 샤드(Shard)
    + 정의: 인덱스의 데이터를 나누는 단위입니다. 인덱스는 여러 샤드로 분할될 수 있으며, 각 샤드는 고유한 데이터 조각을 포함합니다.
    + 역할: 데이터의 분산 저장과 병렬 처리. 클러스터의 확장성과 성능을 향상시키기 위해 사용됩니다.
        + Primary Shard: 인덱스의 기본 샤드. 모든 문서는 기본 샤드에 저장됩니다.
        + Replica Shard: 기본 샤드의 복제본. 데이터 가용성과 장애 조치를 위해 사용됩니다.
4. 마스터 노드(Master Node)
    + 정의: 클러스터의 중앙 관리 노드입니다.
    + 역할: 클러스터의 상태를 관리하고, 노드 간의 조정, 인덱스 생성/삭제, 클러스터 설정 변경 등을 수행합니다. 안정성을 위해 마스터 노드는 여러 개의 후보 노드로 구성될 수 있습니다.
5. 데이터 노드(Data Node)
    + 정의: 데이터를 저장하고 검색 요청을 처리하는 노드입니다.
    + 역할: 데이터의 실제 저장 및 검색 요청 처리. 대부분의 클러스터 자원(디스크, CPU, 메모리)을 사용하여 데이터 관리와 쿼리 처리를 수행합니다.
6. 코디네이팅 노드(Coordinating Node)
    + 정의: 클라이언트 요청을 받아 다른 노드로 라우팅하는 노드입니다. 특별한 역할을 하는 노드가 아닌, 모든 노드는 코디네이팅 노드 역할을 할 수 있습니다.
    + 역할: 클라이언트 요청을 적절한 데이터 노드로 라우팅하고, 검색 결과를 통합하여 클라이언트에 반환. 대규모 클러스터에서 부하 분산을 위해 중요합니다.
7. 인제스트 노드(Ingest Node)
    + 정의: 데이터 삽입 시 전처리 파이프라인을 수행하는 노드입니다.
    + 역할: 데이터 삽입 시 필요한 변환, 필터링, 구조 변경 등을 수행하여 데이터가 저장되기 전에 전처리 과정을 거치도록 합니다.
8. 머신 러닝 노드(Machine Learning Node)
    + 정의: 머신 러닝 작업을 전담하는 노드입니다.
    + 역할: 이상 탐지, 예측 분석 등의 머신 러닝 작업을 수행. Elastic Search의 머신 러닝 기능을 활용하여 데이터를 분석하고 인사이트를 도출합니다.

### 요약
Elastic Search 클러스터는 여러 노드와 샤드로 구성되어 있으며, 각 구성 요소는 다음과 같은 역할을 합니다:

+ 클러스터: 데이터의 분산 저장 및 관리.
+ 노드: 데이터를 저장하고 검색 요청을 처리.
+ 샤드: 인덱스의 데이터 분산 저장.
+ 마스터 노드: 클러스터 상태 관리.
+ 데이터 노드: 데이터 저장 및 검색 요청 처리.
+ 코디네이팅 노드: 클라이언트 요청 라우팅.
+ 인제스트 노드: 데이터 전처리.
+ 머신 러닝 노드: 머신 러닝 작업 수행.

------------

## Elastic Search CRUD
+ 예시데이터

``` json
{
  "index": "employees",
  "document": {
    "_id": "1",
    "name": "Alice",
    "position": "Engineer",
    "age": 30
  }
}
{
  "index": "employees",
  "document": {
    "_id": "2",
    "name": "Bob",
    "position": "Manager",
    "age": 40
  }
}

```

### Create

``` bash
# Alice 문서 생성
curl -X POST "localhost:9200/employees/_doc/1" -H 'Content-Type: application/json' -d'
{
  "name": "Alice",
  "position": "Engineer",
  "age": 30
}
'

# Bob 문서 생성
curl -X POST "localhost:9200/employees/_doc/2" -H 'Content-Type: application/json' -d'
{
  "name": "Bob",
  "position": "Manager",
  "age": 40
}
'

```

### Read

``` bash
# Alice 문서 조회
curl -X GET "localhost:9200/employees/_doc/1"

# Bob 문서 조회
curl -X GET "localhost:9200/employees/_doc/2"

```

### Update

``` bash
# Alice 문서의 나이 수정
curl -X POST "localhost:9200/employees/_update/1" -H 'Content-Type: application/json' -d'
{
  "doc": {
    "age": 31
  }
}
'

# Bob 문서의 직책 수정
curl -X POST "localhost:9200/employees/_update/2" -H 'Content-Type: application/json' -d'
{
  "doc": {
    "position": "Senior Manager"
  }
}
'
```

### Delete

``` bash
# Alice 문서 삭제
curl -X DELETE "localhost:9200/employees/_doc/1"

# Bob 문서 삭제
curl -X DELETE "localhost:9200/employees/_doc/2"

```

## Elastic Search VS RDBMS

### 데이터 CRUD 방식의 차이
RDBMS의 경우 CRUD을 위해 클라이언트에서 관계형 DB서버에 연결을 하고 SQL을 보내는 방식입니다.

(SELECT, INSERT, DELETE, UPDATE등)

Elasticsearch에서는 Restful API를 사용합니다. HTTP 통신에서 사용하는 GET, POST, PUT, DELETE등의 메소드가 그대로 적용되어 사용됩니다. 하지만 데이터 삽입인 POST의 경우 RDBMS와 다른 특성을 가지는데, 스키마가 미리 저장되어 있지 않더라도 자동으로 필드를 생성하고 저장한다는 점입니다. 이러한 특징은 큰 유연성을 제공하지만 또 다른 문제를 야기할 수 있습니다.

![ElasticSearch](/assets/img/post/2024-06-27-elastic-search/1.png)



### 용어 비교
![ElasticSearch](/assets/img/post/2024-06-27-elastic-search/2.png)

+ Elasticsearch에서 인덱스는 위에서 볼 수 있듯 RDBMS의 Database와 같은 개념입니다. 하지만 Elasticsearch에서는 서로 다른 데이터베이스에서도 검색할 필드명이 존재한다면 여러 개의 데이터베이스를 한번에 조회할 수 있습니다.
+ 기존 RDBMS는 스키마라는 구조에 따라 데이터를 적합한 형태로 변형하여 저장 및 관리하지만, Elasticsearch는 비정형의 다양한 형태의 문서도 자동으로 색인, 검색이 가능하다는 특징이 있습니다.