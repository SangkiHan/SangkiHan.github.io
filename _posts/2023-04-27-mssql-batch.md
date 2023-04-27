---
title: MSSQL 등록되어 있는 배치조회
author: SangkiHan
date: 2023-04-26 13:50:00 +0900
categories: [DB, MSSQL]
tags: [DB]
---

------------

#### 현재 MSSQL에서 등록되어 있는 배치작업들을 확인한다.

##### SQL
``` sql
    SELECT
	A.job_id,  -- 배치ID
    A.name, -- 배치이름
    A.enabled, -- 사용여부
    description, -- 배치설명
    D.command,  -- 배치실행시 실행하는 명령어
    freq_type,
    freq_interval,
    STUFF(
        STUFF(RIGHT('000000' + CAST([active_start_time] AS VARCHAR(6)), 6)
                    , 3, 0, ':')
                , 6, 0, ':'),   -- 배치작업시간
    A.date_created,     -- 배치 등록기간
    A.date_modified,   -- 배치 수정기간
	D.step_id, -- 배치가 실행되는 순서
    D.step_name, -- 배치설명
    D.subsystem
    FROM msdb.dbo.sysjobs A 
        INNER JOIN msdb.dbo.sysjobschedules B ON A.job_id = B.job_id 
        INNER JOIN msdb.dbo.sysschedules C ON B.schedule_id = C.schedule_id 
        INNER JOIN msdb.dbo.sysjobsteps D ON A.job_id = D.job_id 
    ORDER BY A.name
```
#### freq_type
-   1 : 1번
-   4 : 매일
-   8 : 매주
-   16 : 매월
-   32 : 매월
-   64 : SQL이 실행될 때 마다
-   128 : 서버가 유휴 할 때 마다.

#### freq_interval
-   1 : 일
-   2 : 월
-   3 : 화
-   4 : 수
-   5 : 목
-   6 : 금
-   7 : 토
-   8 : 일
-   9 : 평일
-   10 : 주말


