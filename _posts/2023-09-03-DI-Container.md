---
title: DI Container
author: SangkiHan
date: 2023-09-03 11:20:00 +0900
categories: [Spring]
tags: [Java]
---
------------

## DI(Dependency Injection)
```DI(의존성주입)```이란 외부에서 실제 구현 객체를 생성하고 Service단에서 사용시 Service에 전달하여 구현 객체와의 의존관계를 연결해주는 것을 의존성 주입이라 한다.  
말로는 이해가 잘 가지 않으니 아래 코드를 확인해보자.  
  
MemberServiceImpl.class에서 MemberRepository.class에서를 의존하고 있다. MemberRepository.class라는 interface를 의존하지만 MemoryMemberRepository.class 구현체를 의존하지 않고 있다.  
만약 아래처럼 MemoryMemberRepository.class 구현체까지 의존한다면 구현체가 수정 될 시 MemberService.class 코드를 수정해야 하기 때문에 SOLID원칙에서 DIP원칙을 위반하게 되기 때문에 좋지 않은 방법이기 때문에 MemberRepository.class라는 interface만 의존하는 것이 좋은 방법이다.  

``` java
private final MemberRepository memberRepository = new MemoryMemberRepository();
```

``` java
public class MemberServiceImpl implements MemberService{
	
	private final MemberRepository memberRepository;
	
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

그렇기 때문에 구현체 역할을 지정해주는 AppConfig.class 만들어 AppConfig.class에게 구현체 의존에 대한 역할을 부여한다.  
이렇게 역할을 부여해줄 시 구현체가 변경될 시 MemberService.class의 코드는 전혀 건들지 않고 AppConfig.class만 부여하면 되기 때문에  
MemberSerive.class와 AppConfig.class의 역할이 정확히 분리되어 DIP원칙을 방지하고 유지보수에도 큰 강점이 생긴다.
``` java
@Configuration
public class AppConfig {
	@Bean
	public MemberService memberService() {
		return new MemberServiceImpl(memberRepository());
	}
}
```

그리고 AppConfig.class처럼 구현체의 의존성을 관리하는 공간을 ```DI컨테이너``` 라고 한다.