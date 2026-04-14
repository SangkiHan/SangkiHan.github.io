---
layout: post
title: "Kafka 클러스터 Grafana/Prometheus Consumer Lag 모니터링 구축"
date: 2026-04-10 10:00:00 +0900
categories: [Monitoring, Kafka]
tags: [Monitoring, Kafka, Prometheus, Grafana, ConsumerLag]
---

3대의 Azure VM에 구축한 Kafka KRaft 클러스터를 Grafana/Prometheus로 모니터링하는 방법을 정리한다.

---

## 모니터링 대상

Kafka 클러스터에서 Lag을 모니터링 하려한다.

**Consumer Lag이 중요한 이유:** Lag이 계속 쌓이면 컨슈머가 처리를 못 따라가고 있다는 신호다. 빠르게 감지해야 장애 전에 스케일아웃할 수 있다.

---

## 전체 구성도

```
[VM1: kafka-1] ─── JMX Exporter :9101 ──┐
[VM2: kafka-2] ─── JMX Exporter :9101 ──┼── prometheus-kafka :9091 ── Grafana :3000
[VM3: kafka-3] ─── JMX Exporter :9101 ──┘

[VM1: kafka-exporter :9308] ─┐
[VM2: kafka-exporter :9308] ─┼── prometheus-kafka :9091
[VM3: kafka-exporter :9308] ─┘  (HA 구성 - VM1 장애 시 VM2, VM3이 수집 유지)

모니터링 서버 (별도): Prometheus + Grafana
```

---

## Step 1. JMX Exporter 설정 (각 VM)

JMX Exporter를 Kafka JVM에 Java Agent로 붙여서 브로커 내부 메트릭을 HTTP로 노출한다.

**jar 파일 다운로드 (각 VM에서):**
```bash
wget https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.20.0/jmx_prometheus_javaagent-0.20.0.jar
```

**jmx-exporter-config.yml:**
```yaml
lowercaseOutputName: true
lowercaseOutputLabelNames: true
```

---

## Step 2. docker-compose 설정 (각 VM)

**VM1 docker-compose.yml:**
```yaml
services:
  kafka-1:
    image: confluentinc/cp-kafka:7.6.0
    container_name: kafka-1
    ports:
      - "9092:9092"
      - "9093:9093"
      - "9101:9101"   # JMX Exporter
    environment:
      CLUSTER_ID: 'q1Sh3IspTsyGH5YQFZK1pw'
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://VM1_IP:9092
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@VM1_IP:9093,2@VM2_IP:9093,3@VM3_IP:9093
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "false"
      KAFKA_OPTS: "-javaagent:/opt/jmx-exporter/jmx_prometheus_javaagent-0.20.0.jar=9101:/opt/jmx-exporter/config.yml"
    volumes:
      - kafka1_data:/var/lib/kafka/data
      - ./jmx_prometheus_javaagent-0.20.0.jar:/opt/jmx-exporter/jmx_prometheus_javaagent-0.20.0.jar
      - ./jmx-exporter-config.yml:/opt/jmx-exporter/config.yml

  kafka-exporter:
    image: danielqsj/kafka-exporter:latest
    container_name: kafka-exporter
    ports:
      - "9308:9308"
    command:
      - --kafka.server=VM1_IP:9092
      - --kafka.server=VM2_IP:9092
      - --kafka.server=VM3_IP:9092
    depends_on:
      - kafka-1
    restart: always

volumes:
  kafka1_data:
```

VM2, VM3은 `KAFKA_NODE_ID`, `KAFKA_ADVERTISED_LISTENERS`, 볼륨명만 각 VM에 맞게 변경하면 된다.

**kafka-exporter는 모든 VM에 배포하는 이유:**
VM1이 다운되면 VM1의 kafka-exporter도 같이 내려간다. VM2, VM3에도 동일하게 배포해두면 Consumer Lag 수집이 끊기지 않는다.

---

## Step 3. Prometheus 설정 (모니터링 서버)

기존 Prometheus와 분리해서 Kafka 전용으로 별도 포트(9091)에 띄운다.

**prometheus-kafka.yml:**
```yaml
# 기존 prometheus.yml의 scrape_configs에 아래 항목 추가

scrape_configs:

  # 브로커별 JMX 메트릭 (처리량, ISR, Controller 등)
  - job_name: 'kafka-jmx'
    static_configs:
      - targets:
          - 'VM1_IP:9101'   # VM1 브로커1
          - 'VM2_IP:9101'      # VM2 브로커2
          - 'VM3_IP:9101'   # VM3 브로커3
    relabel_configs:
      - source_labels: [__address__]
        regex: '(.+):\d+'
        target_label: instance

  # Consumer Lag 메트릭 (VM1에서 3개 브로커 전체 수집)
  - job_name: 'kafka-exporter'
    static_configs:
      - targets:
          - 'VM1_IP:9308'
          - 'VM2_IP:9308'
          - 'VM3_IP:9308'
```

**docker-compose.yml (모니터링 서버):**
```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=15d'
      - '--web.enable-lifecycle'

  prometheus-kafka:
    image: prom/prometheus:latest
    container_name: prometheus-kafka
    ports:
      - "9091:9090"
    volumes:
      - ./prometheus-kafka.yml:/etc/prometheus/prometheus.yml
      - prometheus_kafka_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=15d'
      - '--web.enable-lifecycle'

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_USER=your_admin_id
      - GF_SECURITY_ADMIN_PASSWORD=your_admin_password
    depends_on:
      - prometheus
      - prometheus-kafka

volumes:
  prometheus_data:
  prometheus_kafka_data:
  grafana_data:
```

---

## Step 4. Grafana 데이터소스 추가

1. Grafana 접속 → 좌측 메뉴 **Connections → Data sources**
2. **Add new data source → Prometheus**
3. 설정:
   ```
   Name: prometheus-kafka
   URL:  http://prometheus-kafka:9090
   ```
   (같은 docker-compose 네트워크라면 컨테이너명으로 연결)
4. **Save & test** → `Successfully queried the Prometheus API` 확인

---

## Step 5. 대시보드 임포트

커뮤니티 대시보드를 그대로 임포트하면 데이터소스가 `DS_MARI`로 하드코딩되어 있어서 No data가 뜬다. JSON을 수정해서 임포트해야 한다.

**대시보드 JSON 다운로드 및 데이터소스 치환:**
```bash
# Consumer Lag 대시보드 (ID: 7589)
curl -o kafka-lag.json https://grafana.com/api/dashboards/7589/revisions/latest/download
sed -i 's/DS_MARI/prometheus-kafka/g' kafka-lag.json
```

**임포트:**
Grafana → Dashboards → New → Import → Upload JSON file

---

## Step 6. 수집 확인

**각 VM에서 JMX 메트릭 확인:**
```bash
curl -s http://localhost:9101/metrics | grep -E "kafka_server_broker|kafka_controller"
```

정상 응답 예시:
```
kafka_server_broker_topic_bytesinpersec_rate 0.0
kafka_server_broker_topic_messagesinpersec_rate 0.0
kafka_controller_active_count 0.0   # Controller가 아닌 브로커는 0
```
![Grafana](/assets/img/post/2026-04-10-kafka-grafana-monitoring-setup/1.png)

**Prometheus targets 확인:**
```bash
curl -s http://localhost:9091/api/v1/targets | python3 -m json.tool | grep -E '"health"|"scrapeUrl"'
```

3개 VM 모두 `"health": "up"` 이어야 한다.

---

## 핵심 포인트

1. **Java Agent 방식** 사용 - 별도 컨테이너 방식은 RMI 프로토콜 특성상 hostname 문제가 발생하기 쉽다
2. **kafka-exporter는 모든 VM에 배포** - HA 구성으로 Consumer Lag 수집 끊김 방지
3. **Prometheus를 용도별로 분리** - 기존 인프라 모니터링과 Kafka 모니터링을 분리하면 관리가 편하다
4. **대시보드 JSON 직접 수정** - DS_PROMETHEUS 못 읽는 이슈 방지
