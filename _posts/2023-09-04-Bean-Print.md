---
title: Spring에 등록된 모든 Bean 출력하기
author: SangkiHan
date: 2023-09-04 11:20:00 +0900
categories: [Java, Spring]
tags: [Java]
---
------------

Spring Container에 Bean이 정상적으로 등록이 되었는지 어떤 구현체가 의존되어 있는 지 확인해야할 때가 있을 수 있다.  
Bean주입은 실습을 해야하기에 AppConfig.class를 통해 Bean등록을 하였다.  

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

#### ApplicationContextInfoTest.class
Spring에 등록되어 있는 모든 Bean리스트 출력
``` java
public class ApplicationContextInfoTest {
	AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig.class);
	
	@Test
	@DisplayName("모든 빈 출력하기")
	void findAllBean() {
		String[] beanDefinitionNames = applicationContext.getBeanDefinitionNames();
		for(String beanDefinitionName : beanDefinitionNames) {
			Object bean = applicationContext.getBean(beanDefinitionName);
			System.out.println("name = "+beanDefinitionName+" object = " + bean);
		}
	}
}
```

출력
``` text
name = org.springframework.context.annotation.internalConfigurationAnnotationProcessor object = org.springframework.context.annotation.ConfigurationClassPostProcessor@294e5088
name = org.springframework.context.annotation.internalAutowiredAnnotationProcessor object = org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor@51972dc7
name = org.springframework.context.annotation.internalCommonAnnotationProcessor object = org.springframework.context.annotation.CommonAnnotationBeanPostProcessor@3700ec9c
name = org.springframework.context.event.internalEventListenerProcessor object = org.springframework.context.event.EventListenerMethodProcessor@2002348
name = org.springframework.context.event.internalEventListenerFactory object = org.springframework.context.event.DefaultEventListenerFactory@5911e990
name = appConfig object = com.spring.AppConfig$$EnhancerBySpringCGLIB$$65d982fa@31000e60
name = memberService object = com.spring.member.MemberServiceImpl@1d470d0
name = orderService object = com.spring.order.OrderServiceImpl@24d09c1
name = memberRepository object = com.spring.member.MemoryMemberRepository@54c62d71
name = discountPolicy object = com.spring.discount.FixDiscountPolicy@65045a87
```

Spring내부에 등록되어있는 Bean들을 제외한 애플리케이션 Bean출력
``` java
public class ApplicationContextInfoTest {
	AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig.class);
	
	@Test
	@DisplayName("애플리케이션 빈 출력하기")
	void findApplicationBean() { 
		String[] beanDefinitionNames = applicationContext.getBeanDefinitionNames();
		for(String beanDefinitionName : beanDefinitionNames) {
			BeanDefinition beanDefinition = applicationContext.getBeanDefinition(beanDefinitionName);
			
			if(beanDefinition.getRole() == BeanDefinition.ROLE_APPLICATION) {
				Object bean = applicationContext.getBean(beanDefinitionName);
				System.out.println("name = " + beanDefinitionName + " object = " + bean);
			}
		}
	}
}
```

출력
``` text
name = appConfig object = com.spring.AppConfig$$EnhancerBySpringCGLIB$$65d982fa@31000e60
name = memberService object = com.spring.member.MemberServiceImpl@1d470d0
name = orderService object = com.spring.order.OrderServiceImpl@24d09c1
name = memberRepository object = com.spring.member.MemoryMemberRepository@54c62d71
name = discountPolicy object = com.spring.discount.FixDiscountPolicy@65045a87
```