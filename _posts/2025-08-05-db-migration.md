---
layout: post
title:  "대용량 데이터 마이그레이션 성능 개선기"
date:   2025-08-05 09:00:00 +0900
categories: [DB, Performance]
tags: [DB, MongoDB, Migration, Performance]
---

# 마이그레이션 일지

전력, 온도 5분데이터를 Mongo로 옮겨야함

점포는 약 17000개 5분마다 데이터가 17000개가 들어오니 하루에 약 4896000건의 데이터

1년의 데이터를 이관을 해야하니 전력, 온도 각각 약 1,787,040,000건의 데이터를 이관을 해야한다.

총 3,574,080,000건

현재 데이터가 한행에 시간당 데이터가 한꺼번에 들어가있다.
하지만 MongoDB에는 5분단위로 쪼개서 넣어야한다.

그렇기 때문에 마이그레이션 툴을 사용하지 않고 Java로 하나하나 파싱하여 insert하기로 결정했다.

```java
public void migrationElecBaseToMongo() {
        long startTime = System.currentTimeMillis();
        if (!elecBaseTemplate.collectionExists("20240801")) {
            elecBaseTemplate.createCollection("20240801");
        }

        IndexOperations indexOps = elecBaseTemplate.indexOps("20240801");
        indexOps.ensureIndex(new Index().on("site_id", Sort.Direction.ASC));
        indexOps.ensureIndex(new Index().on("reportDttm", Sort.Direction.ASC));
        indexOps.ensureIndex(new Index().on("dateIdx", Sort.Direction.ASC));
        indexOps.ensureIndex(new Index().on("hourIdx", Sort.Direction.ASC));

        List<ElecBaseDto> elecBaseDtoList = elecBaseMapper.retrieveElecBaseData();
        List<ElecBase> elecBaseListDto = new ArrayList<>();

        for (ElecBaseDto dto : elecBaseDtoList) {
            elecBaseListDto.addAll(createElecBaseList(dto));
            elecBaseTemplate.insert(createElecBaseList(dto), "20240801");
        }

        long endTime = System.currentTimeMillis();
        long duration = endTime - startTime;

        logger.info("=== ElecBase 마이그레이션 완료 ===");
        logger.info("총 소요시간: {}ms ({}초)", duration, (duration / 1000.0));
        logger.info("처리된 원본 데이터: {}건", elecBaseDtoList.size());
        logger.info("생성된 ElecBase: {}건", elecBaseListDto.size());
    }
```

1시간씩 데이터를 조회하고 하나하나 파싱 후 바로 Mongo에 각가 insert를 하니 아래와 같은 성능이 나왔다.

=== ElecBase 마이그레이션 완료 ===  
총 소요시간: 1682151ms (1682.151초)  
처리된 원본 데이터: 383138건  
생성된 ElecBase: 4597656건

1시간 데이터를 넣는데 약 30분… 너무 길다

```java
public void migrationElecBaseToMongo() {
        long startTime = System.currentTimeMillis();
        if (!elecBaseTemplate.collectionExists("20240802")) {
            elecBaseTemplate.createCollection("20240802");
        }

        IndexOperations indexOps = elecBaseTemplate.indexOps("20240802");
        indexOps.ensureIndex(new Index().on("site_id", Sort.Direction.ASC));
        indexOps.ensureIndex(new Index().on("reportDttm", Sort.Direction.ASC));
        indexOps.ensureIndex(new Index().on("dateIdx", Sort.Direction.ASC));
        indexOps.ensureIndex(new Index().on("hourIdx", Sort.Direction.ASC));

        List<ElecBaseDto> elecBaseDtoList = elecBaseMapper.retrieveElecBaseData();
        List<ElecBase> elecBaseListDto = new ArrayList<>();

        for (ElecBaseDto dto : elecBaseDtoList) {
            elecBaseListDto.addAll(createElecBaseList(dto));
        }

        elecBaseTemplate.insert(elecBaseListDto, "20240802");

        long endTime = System.currentTimeMillis();
        long duration = endTime - startTime;

        logger.info("=== ElecBase 마이그레이션 완료 ===");
        logger.info("총 소요시간: {}ms ({}초)", duration, (duration / 1000.0));
        logger.info("처리된 원본 데이터: {}건", elecBaseDtoList.size());
        logger.info("생성된 ElecBase: {}건", elecBaseListDto.size());
    }
```

```shell
com.mongodb.MongoInternalException: Unexpected exception
	at com.mongodb.internal.connection.InternalStreamConnection.translateReadException(InternalStreamConnection.java:711) ~[mongodb-driver-core-4.8.2.jar:na]
	at com.mongodb.internal.connection.InternalStreamConnection.receiveMessageWithAdditionalTimeout(InternalStreamConnection.java:577) ~[mongodb-driver-core-4.8.2.jar:na]
	at com.mongodb.internal.connection.InternalStreamConnection.receiveCommandMessageResponse(InternalStreamConnection.java:413) ~[mongodb-driver-core-4.8.2.jar:na]
	at com.mongodb.internal.connection.InternalStreamConnection.receive(InternalStreamConnection.java:372) ~[mongodb-driver-core-4.8.2.jar:na]
	at com.mongodb.internal.connection.DefaultServerMonitor$ServerMonitorRunnable.lookupServerDescription(DefaultServerMonitor.java:226) ~[mongodb-driver-core-4.8.2.jar:na]
	at com.mongodb.internal.connection.DefaultServerMonitor$ServerMonitorRunnable.run(DefaultServerMonitor.java:158) ~[mongodb-driver-core-4.8.2.jar:na]
	at java.base/java.lang.Thread.run(Thread.java:842) ~[na:na]
Caused by: java.lang.OutOfMemoryError: Java heap space
```

mongo에 insert하는 부분을 한꺼번에 하도록 수정했지만 java heap메모리가 터져버리는 이슈가 생겼다

```java
public void migrationElecBaseToMongo() {
        long startTime = System.currentTimeMillis();
        if (!elecBaseTemplate.collectionExists("20240802")) {
            elecBaseTemplate.createCollection("20240802");
        }

        IndexOperations indexOps = elecBaseTemplate.indexOps("20240802");
        indexOps.ensureIndex(new Index().on("site_id", Sort.Direction.ASC));
        indexOps.ensureIndex(new Index().on("reportDttm", Sort.Direction.ASC));
        indexOps.ensureIndex(new Index().on("dateIdx", Sort.Direction.ASC));
        indexOps.ensureIndex(new Index().on("hourIdx", Sort.Direction.ASC));

        List<ElecBaseDto> elecBaseDtoList = elecBaseMapper.retrieveElecBaseData();
        List<ElecBase> elecBaseListDto = new ArrayList<>();

        for (ElecBaseDto dto : elecBaseDtoList) {
            elecBaseListDto.addAll(createElecBaseList(dto));
        }

        // 만 건씩 청크 단위로 insert
        int chunkSize = 10000;
        int totalSize = elecBaseListDto.size();
        int batchCount = 0;

        for (int i = 0; i < totalSize; i += chunkSize) {
            int endIndex = Math.min(i + chunkSize, totalSize);
            List<ElecBase> chunk = elecBaseListDto.subList(i, endIndex);

            elecBaseTemplate.insert(chunk, "20240802");
            batchCount++;

            logger.info("배치 {} 완료: {}건 처리 ({}/{})", batchCount, chunk.size(), endIndex, totalSize);
        }

        long endTime = System.currentTimeMillis();
        long duration = endTime - startTime;

        logger.info("=== ElecBase 마이그레이션 완료 ===");
        logger.info("총 소요시간: {}ms ({}초)", duration, (duration / 1000.0));
        logger.info("처리된 원본 데이터: {}건", elecBaseDtoList.size());
        logger.info("생성된 ElecBase: {}건", elecBaseListDto.size());
    }
```

10000번씩 자르도록 하였다. 
10000번씩 잘라 Mongo와의 커넥션을 최소화 하였더니 아래와 같은 성능향상이 있었다
약 5배 성능개선

=== ElecBase 마이그레이션 완료 ===
총 소요시간: 362439ms (362.439초)
처리된 원본 데이터: 383603건
생성된 ElecBase: 4603236건

```java
public void migrationElecBaseToMongo() {
        long startTime = System.currentTimeMillis();
        if (!elecBaseTemplate.collectionExists("20240802")) {
            elecBaseTemplate.createCollection("20240802");
        }

        IndexOperations indexOps = elecBaseTemplate.indexOps("20240802");
        indexOps.ensureIndex(new Index().on("site_id", Sort.Direction.ASC));
        indexOps.ensureIndex(new Index().on("reportDttm", Sort.Direction.ASC));
        indexOps.ensureIndex(new Index().on("dateIdx", Sort.Direction.ASC));
        indexOps.ensureIndex(new Index().on("hourIdx", Sort.Direction.ASC));

        List<ElecBaseDto> elecBaseDtoList = elecBaseMapper.retrieveElecBaseData();
        List<ElecBase> elecBaseListDto = new ArrayList<>();

        for (ElecBaseDto dto : elecBaseDtoList) {
            elecBaseListDto.addAll(createElecBaseList(dto));
        }

        // 10만 건씩 청크 단위로 insert
        int chunkSize = 100000;
        int totalSize = elecBaseListDto.size();
        int batchCount = 0;

        for (int i = 0; i < totalSize; i += chunkSize) {
            int endIndex = Math.min(i + chunkSize, totalSize);
            List<ElecBase> chunk = elecBaseListDto.subList(i, endIndex);

            elecBaseTemplate.insert(chunk, "20240802");
            batchCount++;

            logger.info("배치 {} 완료: {}건 처리 ({}/{})", batchCount, chunk.size(), endIndex, totalSize);
        }

        long endTime = System.currentTimeMillis();
        long duration = endTime - startTime;

        logger.info("=== ElecBase 마이그레이션 완료 ===");
        logger.info("총 소요시간: {}ms ({}초)", duration, (duration / 1000.0));
        logger.info("처리된 원본 데이터: {}건", elecBaseDtoList.size());
        logger.info("생성된 ElecBase: {}건", elecBaseListDto.size());
    }
```

10 만건을 해도 성능향상은 없다.

=== ElecBase 마이그레이션 완료 ===
총 소요시간: 394610ms (394.61초)
처리된 원본 데이터: 384067건
생성된 ElecBase: 4608804건

그럼 뭐로 시간을 줄일수있을까?

현재 mongodb에 동기로 insert를 해주고있다. 

하지만 데이터의 순서는 전혀 중요하지 않기때문에 비동기로 넣어도 되지 않을까? 생각했다

```java
public void migrationElecBaseToMongo() {
        long startTime = System.currentTimeMillis();
        if (!elecBaseTemplate.collectionExists("20240806")) {
            elecBaseTemplate.createCollection("20240806");
        }

        IndexOperations indexOps = elecBaseTemplate.indexOps("20240806");
        indexOps.ensureIndex(new Index().on("site_id", Sort.Direction.ASC));
        indexOps.ensureIndex(new Index().on("reportDttm", Sort.Direction.ASC));
        indexOps.ensureIndex(new Index().on("dateIdx", Sort.Direction.ASC));
        indexOps.ensureIndex(new Index().on("hourIdx", Sort.Direction.ASC));

        List<ElecBaseDto> elecBaseDtoList = elecBaseMapper.retrieveElecBaseData();
//        List<ElecBase> elecBaseListDto = new ArrayList<>();

//        for (ElecBaseDto dto : elecBaseDtoList) {
//            elecBaseListDto.addAll(createElecBaseList(dto));
//        }
        List<ElecBase> elecBaseListDto = elecBaseDtoList.parallelStream()
                .flatMap(dto -> createElecBaseList(dto).stream())
                .toList();

        int chunkSize = 10000;
        int totalSize = elecBaseListDto.size();

        // 스레드 풀 생성 (CPU 코어 수에 맞춰 조정)
        ThreadPoolExecutor executor = (ThreadPoolExecutor) Executors.newFixedThreadPool(
                Math.min(8, Runtime.getRuntime().availableProcessors())
        );

        List<Future<?>> futures = new ArrayList<>();

        for (int i = 0; i < totalSize; i += chunkSize) {
            final int startIndex = i;
            final int endIndex = Math.min(i + chunkSize, totalSize);
            final int batchNumber = (i / chunkSize) + 1;

            Future<?> future = executor.submit(() -> {
                List<ElecBase> chunk = elecBaseListDto.subList(startIndex, endIndex);
                elecBaseTemplate.insert(chunk, "20240806");
                logger.info("배치 {} 완료: {}건 처리", batchNumber, chunk.size());
            });

            futures.add(future);
        }

        // 모든 작업 완료 대기
        for (Future<?> future : futures) {
            try {
                future.get();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }

        executor.shutdown();

        long endTime = System.currentTimeMillis();
        long duration = endTime - startTime;

        logger.info("=== ElecBase 마이그레이션 완료 ===");
        logger.info("총 소요시간: {}ms ({}초)", duration, (duration / 1000.0));
        logger.info("처리된 원본 데이터: {}건", elecBaseDtoList.size());
        logger.info("생성된 ElecBase: {}건", elecBaseListDto.size());
    }
```

460만건의 데이터를 이관하는데 약 2분이 걸렸다. 3배의 성능향상

=== ElecBase 마이그레이션 완료 ===
총 소요시간: 140352ms (140.352초)
처리된 원본 데이터: 385165건
생성된 ElecBase: 4621980건


## 마무리

이번 대용량 데이터 마이그레이션 작업을 통해 몇 가지 중요한 성능 개선 원칙을 다시 한번 확인할 수 있었습니다.

- **배치(Batch) 처리의 중요성**: 대용량 Insert 작업 시, 단일 처리보다 배치 처리가 네트워크 및 DB 부하를 줄여 성능에 훨씬 유리합니다. 단순히 모든 데이터를 메모리에 모아 한 번에 처리하는 것은 `OutOfMemoryError`를 유발할 수 있으므로, 적절한 크기로 나누어 처리하는 **청크(Chunk) 기반 배치**가 효과적이었습니다.

- **비동기 및 병렬 처리의 힘**: 데이터의 순서가 중요하지 않은 경우, **병렬 처리(Parallel Processing)**와 **비동기(Asynchronous)** 작업은 I/O 대기 시간을 크게 줄여줍니다. Java의 `parallelStream`과 `ThreadPoolExecutor`를 활용하여 약 3배의 추가적인 성능 향상을 이끌어낼 수 있었습니다.

- **데이터베이스 인덱싱의 기본**: 애플리케이션 로직 최적화만큼 **데이터베이스 인덱싱**의 중요성을 다시 한번 깨달았습니다. 쿼리가 예상과 다르게 동작할 때는 실행 계획을 확인하고, 복합 인덱스를 효과적으로 활용하는 방안을 찾는 것이 중요합니다.

결론적으로, 대용량 데이터를 다룰 때는 메모리 관리, I/O 최적화, 그리고 데이터베이스와의 상호작용을 모두 고려하는 종합적인 접근이 필수적입니다.
