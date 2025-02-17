---
title: 싱글톤(Singleton) 패턴
author: SangkiHan
date: 2023-09-05 11:20:00 +0900
categories: [Spring]
tags: [Java]
---
------------

# 싱글톤(Singleton) 패턴이란?

싱글톤(Singleton) 패턴의 정의는 단순하다. 객체의 인스턴스가 오직 1개만 생성되는 패턴을 의미한다.  
웹어플리케이션은 하루에 수백 수만번의 요청이 들어올 수 있다.  
그렇다면 아래 코드가 한번에 100번이 호출된다면 어떻게 될까?  
MemoryMemberRepository.class 객체가 100번이 생성될 것 이다.  
그렇다면 100번의 객체가 메모리에 쌓이게 되고 메모리의 낭비가 생길 것이다.  
그렇기 때문에 객체 인스턴스를 딱 한번만 생성하고 가져다 쓰는 것이 훨씬 효율적일 것이다.  
이것이 바로 싱글톤(Singleton) 패턴이다.  

``` java
public class MemberServiceImpl implements MemberService{
	
	private final MemberRepository memberRepository = new MemoryMemberRepository();
	
	public MemberServiceImpl(MemberRepository memberRepository) {
		this.memberRepository = memberRepository;
	}
	
	@Override
	public void join(Member member) {
		memberRepository.save(member);
	}

	@Override
	public Member findMember(Long memberId) {
		return memberRepository.findById(memberId);
	}

}
```

------------

## 테스트코드

Spring 컨테이너에서 실제로 싱글톤을 유지해주는지 아래 코드로 테스트 해보았다.

------------

### 직접 의존성 주입을 한 테스트

AppConfig.class
``` java
public class AppConfig {
	
	public MemberService memberService() {
		return new MemberServiceImpl(memberRepository());
	}

	public OrderService orderService() {
		return new OrderServiceImpl(memberRepository(), discountPolicy());
	}

	public MemberRepository memberRepository() {
		return new MemoryMemberRepository();
	}

	public DiscountPolicy discountPolicy() {
		return new FixDiscountPolicy();
	}

}
```

Spring 컨테이너에서 객체를 가져오지 않고 직접 의존성을 주입해줄시 아래 출력과 같이 두 객체가 상이 하다는 것을 알 수 있다.

SingleToneTest.class

``` java
@Test
@DisplayName("스프링 없는 순수한 DI 컨테이너")
void pureContainer() {
    AppConfig appConfig = new AppConfig();
    
    MemberService memberService1 = appConfig.memberService();
    MemberService memberService2 = appConfig.memberService();
    
    System.out.println(memberService1);
    System.out.println(memberService2);
    
    Assertions.assertThat(memberService1).isSameAs(memberService2);
}
```

``` text
com.spring.member.MemberServiceImpl@31304f14
com.spring.member.MemberServiceImpl@34a3d150
```

------------

### Spring 컨테이너를 활용한 의존성 주입 테스트
AppConfig.class
``` java
@Configuration
public class AppConfig {
	
	@Bean
	public MemberService memberService() {
		return new MemberServiceImpl(memberRepository());
	}
	@Bean
	public OrderService orderService() {
		return new OrderServiceImpl(memberRepository(), discountPolicy());
	}
	@Bean
	public MemberRepository memberRepository() {
		return new MemoryMemberRepository();
	}
	@Bean
	public DiscountPolicy discountPolicy() {
		return new FixDiscountPolicy();
	}

}
```

Spring 컨테이너를 활용하여 의존성을 주입할 시 두 객체가 동일한 것을 확인 할 수 있다.

SingleToneTest.class

``` java
@Test
@DisplayName("스프링을 활용한 DI 컨테이너")
void springContainer() {
    ApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig.class);
    
    MemberService memberService1 = applicationContext.getBean("memberService", MemberService.class);
    MemberService memberService2 = applicationContext.getBean("memberService", MemberService.class);
    
    System.out.println(memberService1);
    System.out.println(memberService2);
    
    Assertions.assertThat(memberService1).isSameAs(memberService2);
}
```

``` text
com.spring.member.MemberServiceImpl@5f6722d3
com.spring.member.MemberServiceImpl@5f6722d3
```

------------
## 주의점
1. 특정 클라이언트에 의존적인 필드 X
2. 특정 클라리언트가 값을 변경할 수 있는 필드가 있으면 X
3. 가급적 읽기만 가능해야함
4. 필드 대신에 자바에서 공유되지 않는, 지역변수, 파라미터 등을 사용해야함

### 테스트코드

아래와 같이 WarningService에 num변수를 변경이 가능하게 테스트 Class를 구현하였다.  

``` java
public class WarningService {
	
	private int num;
	
	public WarningService() {
		
	}
	public WarningService(int num) {
		this.num = num;
	}
	public int getNum() {
		return num;
	}
	public void setNum(int num) {
		this.num = num;
	}
}
```

SingleToneTest.class
``` java
class SingleToneTest {
	@Test
	@DisplayName("싱글톤 패턴 주의점 테스트")
	void singleToneWarningTest() {
		ApplicationContext applicationContext = new AnnotationConfigApplicationContext(Config.class);
		
		WarningService warning1 = applicationContext.getBean("warningService", WarningService.class);
		warning1.setNum(10);
		WarningService warning2 = applicationContext.getBean("warningService", WarningService.class);
		warning2.setNum(20);
		
		System.out.println(warning1.getNum());
		System.out.println(warning2.getNum());
	}
		
	static class Config {
		
		@Bean
		public WarningService warningService() {
			return new WarningService();
		}
	}
}
```

출력
``` text
20
20
```

일반적인 생각으로는 warning1객체에는 10, warning2객체에는 20으로 세팅을 했으니 출력도 10, 20으로 되어야한다고 생각을 했을 것이다.  
하지만 싱글톤 패턴은 처음에 객체를 Spring 컨테이너에 저장을 해두고 가져다가 쓰는 형태이기 때문에 num처럼 전역변수로 값을 변할 수 있는 변수를 쓴다면  
warning2에서 20을 세팅했을 때 warning1이 미리 세팅 해놓은 10을 덮어 버리기 때문에 운영 상황일 경우 큰 장애로 이어 질 수 있기 때문에 주의하여 활용하여야 한다.