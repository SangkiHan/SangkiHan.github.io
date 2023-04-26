---
title: '@Entity'
author: SangkiHan
date: 2023-04-26 13:20:00 +0900
categories: [Spring, JPA]
tags: [JPA]
---

------------
@Entity가 붙은 클래스는 JPA가 관리, Entity라고 한다.  
JPA를 사용하여 테이블과 매핑할 클래스에 @Entity를 붙힌다.

Spring 서버 실행시 Entity를 붙힌 객체클래스의 변수대로 테이블이 생성된 것을 확인할수있다.

------------
### Member.class
``` java
@Getter
@Setter
@Entity
public class Member {
	
	@Id @GeneratedValue
	@Column(name = "member_id")
	private long id;
	private String username;
	private String city;
	private String street;
	private String zipcode;
}
```

### Member Table
![Member table](/assets/img/post/2023-04-26-spring-jpa-entity/memberDB.PNG)