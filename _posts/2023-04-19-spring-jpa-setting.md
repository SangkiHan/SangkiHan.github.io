---
title: Spring JPA 설정
descripions: Spring boot JPA 설정 가이드
author: SangkiHan
date: 2023-04-19 11:20:00 +0900
categories: [ORM, JPA]
tags: [JPA]
---

------------
## 의존성 추가

-   ### Gradle

``` gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    //Spring Security
    implementation 'org.springframework.boot:spring-boot-starter-security'				
    //thymeleaf
    implementation 'org.springframework.boot:spring-boot-starter-thymeleaf'	
    implementation 'nz.net.ultraq.thymeleaf:thymeleaf-layout-dialect'				
    //mysql	
    implementation group: 'mysql', name: 'mysql-connector-java', version: '8.0.32'		
    //jpa
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    
    testImplementation 'org.springframework.boot:spring-boot-starter-test'			
    //lombok	
    compileOnly group: 'org.projectlombok', name: 'lombok', version: '1.18.26'			
} 
```
-   ### Maven

``` xml
<!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-data-jpa -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
    <version>2.7.5</version>
</dependency>
```

------------
##  appication.properties, yml 설정

-   ### appication.properties

``` properties
# true 설정시 JPA 쿼리문 확인 가능
spring.jpa.show-sql=true
# DDL(create, alter, drop) 정의시 DB의 고유 기능을 사용할 수 있다.
# create : 기존 테이블을 삭제하고 새로 생성 [DROP + CREATE]
# create-drop : CREATE 속성에 추가로 어플리케이션을 종료할 때 생성한 DDL을 제거
# update : DB테이블과 엔티티 매핑 정보를 비교해서 변경 사항만 수정 [테이블이 없을 경우 CREATE]
# validate : DB테이블과 엔티티 매핑정보를 비교해서 차이가 있으면 경고를 남기고 어플리케이션을 실행하지 않음
# none : 자동 생성 기능을 사용하지않음
spring.jpa.hibernate.ddl-auto=update
# JPA의 구현체인 Hibernate가 동작하면서 발생한 SQL의 가독성을 높여준다.
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.properties.hibernate.jdbc.exception-handling=ignore

logging.level.org.hibernate.SQL= debug
```

-   ### appication.yml

``` yml
spring:
jpa:
    properties:
    hibernate:
        jdbc:
        exception-handling: ignore
        # JPA의 구현체인 Hibernate가 동작하면서 발생한 SQL의 가독성을 높여준다.
        format_sql: 'true'
    # true 설정시 JPA 쿼리문 확인 가능
    show-sql: 'true'
    hibernate:
    # DDL(create, alter, drop) 정의시 DB의 고유 기능을 사용할 수 있다.
    # create : 기존 테이블을 삭제하고 새로 생성 [DROP + CREATE]
    # create-drop : CREATE 속성에 추가로 어플리케이션을 종료할 때 생성한 DDL을 제거
    # update : DB테이블과 엔티티 매핑 정보를 비교해서 변경 사항만 수정 [테이블이 없을 경우 CREATE]
    # validate : DB테이블과 엔티티 매핑정보를 비교해서 차이가 있으면 경고를 남기고 어플리케이션을 실행하지 않음
    # none : 자동 생성 기능을 사용하지않음
    ddl-auto: update
logging:
level:
    org:
    hibernate:
        SQL: debug
```

