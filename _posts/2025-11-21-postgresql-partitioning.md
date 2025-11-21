---
layout: post
title:  "PostgreSQL 파티셔닝으로 대용량 테이블 성능 개선하기"
date:   2025-11-21 09:00:00 +0900
categories: [DB, Performance]
tags: [PostgreSQL, Partitioning, Performance, pg_partman]
---

# 대용량 날씨 데이터, 파티셔닝으로 해결하다

## 문제 상황

회사 프로젝트에서 전국 지역별 날씨 예보 데이터를 저장하는 `batch.weather_forecast_latest` 테이블을 운영하고 있었다. 시간이 지나면서 데이터가 기하급수적으로 쌓이기 시작했고, 최신 데이터만 필요한 쿼리에서도 **테이블 풀스캔**이 발생하는 심각한 성능 문제가 생겼다.

```sql
-- 기존 테이블 구조
CREATE TABLE batch.weather_forecast_latest (
    report_dttm timestamptz NOT NULL,
    country_cd varchar(50) NOT NULL,
    local_area_code varchar(50) NOT NULL,
    aunnc_time varchar(50) NULL,
    weather_cd varchar(2) NULL,
    forecast_temp numeric(3, 1) NULL,
    forecast_precipitation varchar(3) NULL,
    forecast_wspeed varchar(4) NULL,
    forecast_wdirection varchar(1) NULL,
    forecast_humidity varchar(3) NULL,
    sunrise_time varchar(6) NULL,
    sunset_time varchar(6) NULL,
    regi_id varchar(50) NULL,
    regi_dttm timestamptz NULL,
    final_mod_id varchar(50) NULL,
    final_mod_dttm timestamptz NULL,
    CONSTRAINT weather_forecast_latest_pk PRIMARY KEY (report_dttm, country_cd, local_area_code)
);
```

### 실제 성능 문제

```sql
-- 최근 1주일 데이터 조회 쿼리
EXPLAIN ANALYZE
SELECT * 
FROM batch.weather_forecast_latest
WHERE report_dttm >= NOW() - INTERVAL '7 days'
  AND local_area_code = 'SEOUL';

-- 실행 결과
Seq Scan on weather_forecast_latest  (cost=0.00..245896.00 rows=125 width=120) (actual time=3245.234..12856.891 rows=1008 loops=1)
  Filter: ((report_dttm >= (now() - '7 days'::interval)) AND ((local_area_code)::text = 'SEOUL'::text))
  Rows Removed by Filter: 12458962
Planning Time: 0.458 ms
Execution Time: 12857.234 ms
```

**약 12.8초**나 걸리는 쿼리... 최근 7일 데이터만 필요한데 전체 600만 건을 풀스캔하고 있었다.

## 해결책: Range 파티셔닝

날씨 예보 데이터는 **시계열 특성**이 강하고, 주로 최신 데이터를 조회한다. 또한 오래된 데이터는 주기적으로 삭제해야 하는 요구사항이 있었다. 이런 상황에서는 `report_dttm` 기준 **Range 파티셔닝**이 최적의 선택이었다.

### pg_partman을 선택한 이유

수동으로 파티션을 관리할 수도 있지만, 매월 새로운 파티션을 생성하고 오래된 파티션을 삭제하는 작업을 자동화하기 위해 **pg_partman** Extension을 사용하기로 결정했다.

- 자동 파티션 생성 (미래 파티션 미리 생성)
- 오래된 파티션 자동 삭제
- 기존 데이터 점진적 마이그레이션 (서비스 중단 최소화)

## 구현 과정

### 1. pg_partman 설치

```sql
-- Extension 설치
CREATE SCHEMA IF NOT EXISTS partman;
CREATE EXTENSION IF NOT EXISTS pg_partman SCHEMA partman;
```

### 2. 기존 테이블 백업 및 파티션 테이블 생성

```sql
-- 기존 테이블 백업
ALTER TABLE batch.weather_forecast_latest RENAME TO weather_forecast_latest_old;

-- 파티션 부모 테이블 생성
CREATE TABLE batch.weather_forecast_latest (
    report_dttm timestamptz NOT NULL,
    country_cd varchar(50) NOT NULL,
    local_area_code varchar(50) NOT NULL,
    aunnc_time varchar(50) NULL,
    weather_cd varchar(2) NULL,
    forecast_temp numeric(3, 1) NULL,
    forecast_precipitation varchar(3) NULL,
    forecast_wspeed varchar(4) NULL,
    forecast_wdirection varchar(1) NULL,
    forecast_humidity varchar(3) NULL,
    sunrise_time varchar(6) NULL,
    sunset_time varchar(6) NULL,
    regi_id varchar(50) NULL,
    regi_dttm timestamptz NULL,
    final_mod_id varchar(50) NULL,
    final_mod_dttm timestamptz NULL,
    CONSTRAINT weather_forecast_latest_pk PRIMARY KEY (report_dttm, country_cd, local_area_code)
) PARTITION BY RANGE (report_dttm);

-- 인덱스 생성 (각 파티션에 자동 상속됨)
CREATE INDEX idx_weather_forecast_latest_loc_report 
    ON batch.weather_forecast_latest (local_area_code, report_dttm DESC);
CREATE INDEX weather_forecast_latest_aunnc_time_idx 
    ON batch.weather_forecast_latest (aunnc_time, regi_id);
```

### 3. pg_partman으로 파티션 자동 관리 설정

```sql
-- 파티션 자동 생성 설정
SELECT partman.create_parent(
    p_parent_table := 'batch.weather_forecast_latest',
    p_control := 'report_dttm',
    p_type := 'native',
    p_interval := '1 month',
    p_premake := 3,                    -- 미래 3개월 파티션 미리 생성
    p_start_partition := '2024-11-01'  -- 시작 파티션
);

-- 파티션 유지 정책 설정
UPDATE partman.part_config 
SET 
    retention = '6 months',              -- 6개월 이상 된 파티션 삭제
    retention_keep_table = false,        -- 삭제 시 테이블도 완전 제거
    retention_keep_index = false,
    infinite_time_partitions = true,     -- 미래 파티션 계속 생성
    premake = 3,                         -- 미래 3개월 미리 생성
    optimize_trigger = 4,
    inherit_privileges = true            -- 권한 상속
WHERE parent_table = 'batch.weather_forecast_latest';
```

### 4. 기존 데이터 마이그레이션

```sql
-- 점진적 데이터 마이그레이션 (서비스 영향 최소화)
CALL partman.partition_data_proc(
    p_parent_table := 'batch.weather_forecast_latest',
    p_batch_count := 10000,    -- 한 번에 처리할 배치 크기
    p_batch_interval := 1,     -- 배치 간격(초)
    p_lock_wait := 2           -- 락 대기 시간(초)
);

-- 마이그레이션 진행 상황 확인
SELECT * FROM partman.part_config WHERE parent_table = 'batch.weather_forecast_latest';
```

### 5. 크론잡 설정 (자동 유지보수)

```sql
-- pg_cron Extension으로 매일 파티션 관리 자동 실행
CREATE EXTENSION IF NOT EXISTS pg_cron;

SELECT cron.schedule(
    'partition-maintenance',
    '0 2 * * *',  -- 매일 새벽 2시
    $$CALL partman.run_maintenance_proc()$$
);
```

## 성능 개선 결과

파티셔닝 적용 후 동일한 쿼리를 다시 실행해봤다.

```sql
EXPLAIN ANALYZE
SELECT * 
FROM batch.weather_forecast_latest
WHERE report_dttm >= NOW() - INTERVAL '7 days'
  AND local_area_code = 'SEOUL';

-- 실행 결과
Index Scan using weather_forecast_latest_202511_loc_report_idx 
    on weather_forecast_latest_202511  (cost=0.42..8.45 rows=1 width=120) (actual time=0.234..0.891 rows=1008 loops=1)
  Index Cond: ((local_area_code)::text = 'SEOUL'::text)
  Filter: (report_dttm >= (now() - '7 days'::interval))
Planning Time: 0.156 ms
Execution Time: 0.945 ms
```

### 놀라운 성능 향상

- **실행 시간**: 12,857ms → **0.945ms** (약 **13,600배** 빨라짐!)
- **스캔 방식**: Seq Scan (풀스캔) → Index Scan (인덱스 + 파티션 프루닝)
- **스캔 행 수**: 12,458,962건 → 1,008건

## 기존 쿼리는?

가장 좋았던 점은 **기존 애플리케이션 코드를 전혀 수정하지 않아도 된다**는 것이었다.

```java
// 기존 쿼리 그대로 사용 가능
List<WeatherForecast> forecasts = weatherMapper.selectRecentWeather(
    localAreaCode, 
    LocalDateTime.now().minusDays(7)
);
```

파티션 부모 테이블(`weather_forecast_latest`)을 조회하면 PostgreSQL이 자동으로:
1. WHERE 조건의 `report_dttm`을 분석
2. 해당하는 파티션만 스캔 (**파티션 프루닝**)
3. 각 파티션의 인덱스 활용

## 추가 효과

### 1. 배치 작업 성능 개선

```sql
-- 오래된 데이터 삭제 (기존: 느린 DELETE)
DELETE FROM batch.weather_forecast_latest 
WHERE report_dttm < '2025-05-01';
-- 수백만 건 DELETE: 약 15분 소요

-- 파티셔닝 후: 파티션 DROP (순식간에!)
DROP TABLE batch.weather_forecast_latest_202504;
-- 약 0.5초 소요
```

### 2. 유지보수 편의성

- 파티션별 VACUUM, ANALYZE 가능 (전체 테이블 락 없이)
- 특정 월 데이터만 백업/복구 가능
- 테이블 크기 관리 용이

### 3. 모니터링 개선

```sql
-- 파티션별 데이터 크기 확인
SELECT 
    inhrelid::regclass AS partition_name,
    pg_size_pretty(pg_total_relation_size(inhrelid)) AS size,
    (SELECT count(*) FROM ONLY batch.weather_forecast_latest 
     WHERE tableoid = inhrelid) AS row_count
FROM pg_inherits
WHERE inhparent = 'batch.weather_forecast_latest'::regclass
ORDER BY inhrelid::text;

-- 결과
         partition_name          |  size   | row_count 
---------------------------------+---------+-----------
 weather_forecast_latest_202509  | 425 MB  | 4524160
 weather_forecast_latest_202510  | 428 MB  | 4582320
 weather_forecast_latest_202511  | 356 MB  | 3801240
 weather_forecast_latest_202512  | 0 bytes | 0        (미래 파티션)
```

## 마무리

대용량 시계열 데이터를 다루는 시스템에서 **파티셔닝은 선택이 아닌 필수**라는 것을 다시 한번 깨달았다.

### 파티셔닝을 고려해야 하는 경우

1. **시계열 데이터**로 특정 기간만 주로 조회
2. 테이블 크기가 수천만 건 이상
3. 최신 데이터 위주로 조회하는 패턴
4. 오래된 데이터 주기적 삭제 필요

### 주의사항

- **파티션 키는 변경 불가**: 설계 단계에서 신중하게 선택
- **WHERE 조건에 파티션 키 포함**: 파티션 프루닝 최대한 활용
- **Foreign Key 제약**: 파티션 테이블은 FK에 제한 있음
- **정기 모니터링**: 파티션 자동 생성/삭제가 잘 되고 있는지 확인

pg_partman 덕분에 파티션 관리 부담 없이 **13,600배**의 성능 향상을 얻을 수 있었다. 대용량 데이터로 고민하고 있다면 파티셔닝을 적극 추천한다!
