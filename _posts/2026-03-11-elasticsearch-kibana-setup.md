---
title: Elasticsearch + Kibana Docker 설치 가이드 (with 인증 활성화)
author: SangkiHan
date: 2026-03-11 10:00:00 +0900
categories: [Database, Elasticsearch]
tags: [Elasticsearch, Kibana, Docker, Docker-Compose, Search]
---

## 개요

커뮤니티 기능에 해시태그 검색을 구현하면서 Elasticsearch를 도입했습니다.
MySQL의 `LIKE` 검색은 인덱스를 타지 못해 대규모 데이터에서 느리지만, Elasticsearch는 **역색인(Inverted Index)** 구조로 구글처럼 빠른 검색이 가능합니다.

이 글에서는 Docker Compose로 Elasticsearch와 Kibana를 설치하고, 인증까지 활성화하는 전 과정을 정리합니다.

---

## Elasticsearch란?

**검색 엔진 데이터베이스**입니다.

데이터를 저장할 때 미리 역색인 구조로 인덱싱해두기 때문에 검색 시 전체 테이블을 스캔하지 않습니다.

```
MySQL:  "강아지 산책" 검색 → 전체 테이블 풀스캔 → 느림
ES:     "강아지 산책" 검색 → 역색인 조회        → 매우 빠름
```

**역색인 원리**

```
문서1: "강아지 산책 좋아요"
문서2: "강아지 목욕 힘들어"
문서3: "고양이 산책"

역색인 테이블:
  강아지 → [문서1, 문서2]
  산책   → [문서1, 문서3]
  좋아요 → [문서1]
```

"강아지" 검색 시 역색인에서 바로 문서1, 문서2를 찾아냅니다.

---

## Kibana란?

**Elasticsearch 데이터를 브라우저에서 시각화하는 대시보드 툴**입니다.

| 기능 | 설명 |
|---|---|
| **Dev Tools** | ES에 쿼리를 직접 날려보는 콘솔 |
| **Discover** | 저장된 문서 데이터 탐색 |
| **Dashboard** | 차트/그래프로 데이터 시각화 |
| **Index Management** | 인덱스 생성/삭제/설정 관리 |

---

## Docker Compose 설치

### docker-compose.yml

```yaml
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.12.0
    container_name: elasticsearch
    environment:
      - node.name=es-node
      - cluster.name=puppynote-cluster
      - discovery.type=single-node
      - xpack.security.enabled=true
      - xpack.security.http.ssl.enabled=false
      - xpack.security.transport.ssl.enabled=false
      - ELASTIC_PASSWORD=Puppynote1234!
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
    ports:
      - "9200:9200"
    volumes:
      - esdata:/usr/share/elasticsearch/data
    networks:
      - elastic-net
    healthcheck:
      test: ["CMD-SHELL", "curl -s -u elastic:Puppynote1234! http://localhost:9200/_cluster/health | grep -q 'status'"]
      interval: 30s
      timeout: 10s
      retries: 5

  setup:
    image: curlimages/curl:latest
    container_name: es-setup
    depends_on:
      elasticsearch:
        condition: service_healthy
    networks:
      - elastic-net
    restart: "no"
    entrypoint:
      - curl
      - -s
      - -X
      - POST
      - http://elasticsearch:9200/_security/user/kibana_system/_password
      - -H
      - "Content-Type: application/json"
      - -u
      - elastic:Puppynote1234!
      - -d
      - "{\"password\":\"원하는비밀번호\"}"

  kibana:
    image: docker.elastic.co/kibana/kibana:8.12.0
    container_name: kibana
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
      - ELASTICSEARCH_USERNAME=kibana_system
      - ELASTICSEARCH_PASSWORD=원하는비밀번호
      - SERVER_BASEPATH=/kibana
      - SERVER_REWRITEBASEPATH=true
    ports:
      - "5601:5601"
    depends_on:
      setup:
        condition: service_completed_successfully
    networks:
      - elastic-net

volumes:
  esdata:

networks:
  elastic-net:
    driver: bridge
```

### 인증 관련 환경변수 설명

| 변수 | 설명 |
|---|---|
| `xpack.security.enabled=true` | 인증 활성화 |
| `xpack.security.http.ssl.enabled=false` | HTTP SSL 비활성화 (내부망용) |
| `ELASTIC_PASSWORD` | `elastic` 슈퍼유저 비밀번호 설정 |

---

## kibana_system 비밀번호 문제

여기서 주의해야 할 함정이 있습니다.

`ELASTIC_PASSWORD`는 `elastic` 유저의 비밀번호만 설정합니다.
Kibana가 ES에 접속할 때 사용하는 **`kibana_system` 유저는 별도로 비밀번호를 설정**해야 합니다.

설정하지 않으면 아래와 같은 인증 실패 로그가 발생합니다.

```
Authentication of [kibana_system] was terminated by realm [reserved]
- failed to authenticate user [kibana_system]
```

### 해결 방법

ES가 완전히 기동된 후 아래 명령어로 `kibana_system` 비밀번호를 설정합니다.

```bash
# 1. ES만 먼저 실행
docker-compose up -d elasticsearch

# 2. ES 정상 동작 확인
docker exec elasticsearch curl -s -u elastic:비밀번호 http://localhost:9200

# 3. kibana_system 비밀번호 설정 (딱 한 번만)
docker exec elasticsearch curl -s \
  -X POST http://localhost:9200/_security/user/kibana_system/_password \
  -H "Content-Type: application/json" \
  -u elastic:비밀번호 \
  -d '{"password":"비밀번호"}'

# 응답: {} → 성공

# 4. Kibana 실행
docker-compose up -d kibana
```

---

## Kibana 접속

브라우저에서 접속합니다.

```
http://서버IP:5601
```

로그인 정보:

```
Username: elastic
Password: 설정한 비밀번호
```

---

## Dev Tools에서 쿼리 테스트

접속 후 좌측 메뉴 → **Management → Dev Tools**

```json
# 인덱스 목록 확인
GET /_cat/indices?v

# 전체 문서 조회
GET /posts/_search

# 해시태그 검색
GET /posts/_search
{
  "query": {
    "term": { "hashtags": "강아지산책" }
  }
}

# 인기 해시태그 Top 10 집계
GET /posts/_search
{
  "size": 0,
  "aggs": {
    "top_hashtags": {
      "terms": { "field": "hashtags", "size": 10 }
    }
  }
}
```

---

## Spring Boot 연동 설정

### application.yml

```yaml
elasticsearch:
  host: ${ES_HOST:localhost}
  port: ${ES_PORT:9200}
  username: ${ES_USERNAME:elastic}
  password: ${ES_PASSWORD}
```

### ElasticsearchConfig.java

```java
@Configuration
@ConfigurationProperties(prefix = "elasticsearch")
@Getter @Setter
public class ElasticsearchConfig extends ElasticsearchConfiguration {

    private String host;
    private int port;
    private String username;
    private String password;

    @Override
    public ClientConfiguration clientConfiguration() {
        return ClientConfiguration.builder()
                .connectedTo(host + ":" + port)
                .withBasicAuth(username, password)
                .withConnectTimeout(Duration.ofSeconds(3))
                .withSocketTimeout(Duration.ofSeconds(30))
                .build();
    }
}
```

---

## 환경변수 역할 정리

| 변수 | 사용 위치 | 설명 |
|---|---|---|
| `ELASTIC_PASSWORD` | docker-compose | `elastic` 유저 초기 비밀번호 |
| `KIBANA_PASSWORD` | kibana_system 설정 API | Kibana → ES 접속 비밀번호 |
| `ES_PASSWORD` | Spring Boot | 앱 → ES API 요청 비밀번호 |

세 값은 동일하게 맞춰야 합니다.

---

## 트러블슈팅

### docker-compose.yml YAML 파싱 오류

복사/붙여넣기 시 들여쓰기나 특수문자가 깨지는 경우가 많습니다.
파일을 직접 heredoc으로 작성하면 안전합니다.

```bash
cat > ~/elasticsearch/docker-compose.yml << 'EOF'
# 내용 붙여넣기
EOF
```

### kibana_system 인증 실패

`ELASTIC_PASSWORD`만 설정하고 `kibana_system` 비밀번호를 따로 설정하지 않은 경우입니다.
위의 `kibana_system 비밀번호 문제` 섹션을 참고하세요.

### AWS EC2에서 Kibana 접속 안 될 때

보안그룹 인바운드 규칙에 **5601 포트**가 열려있는지 확인합니다.

```
포트: 5601
소스: 내 IP (전체 오픈은 보안상 비권장)
```
