---
title: Controller Advice 활용
author: SangkiHan
date: 2024-04-30 11:20:00 +0900
categories: [Spring]
tags: [Spring]
---
------------

## Controller Advice
Spring Framework에서 Controller Advice는 전역적으로 발생하는 예외를 처리하고, 요청 및 응답에 대한 공통적인 기능을 제공한다. 이를 통해 코드의 중복을 줄이고, 애플리케이션의 유지보수를 용이하게 만든다.

## 사용사례

기존에는 아래처럼 모든 예외처리를 직접 Controller 로직단에서 처리를 했어야 했다.

## AS-IS
```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/api/v1/auth")
public class LoginController {

	private final LoginService loginService;

	@PostMapping("/login")
	public ApiResponse<LoginResponse> login(@Valid @RequestBody LoginRequest loginRequest) throws
		JsonProcessingException {
		try {
			return ApiResponse.ok(loginService.normalLogin(loginRequest.toServiceRequest()));
		} catch (LuckKidsException e) {
			return ApiResponse.of(
				HttpStatus.BAD_REQUEST,
				e.getMessage(),
				null
			);
		}
	}

	@PostMapping("/oauth/login")
	public ApiResponse<OAuthLoginResponse> oauthLogin(@Valid @RequestBody LoginOauthRequest loginOauthRequest) throws
		JsonProcessingException {
		try {
			return ApiResponse.ok(loginService.oauthLogin(loginOauthRequest.toServiceRequest()));
		} catch (LuckKidsException e) {
			return ApiResponse.of(
				HttpStatus.BAD_REQUEST,
				e.getMessage(),
				null
			);
		}
	}
}
```

아래 처럼 클래스를 생성 후 전역으로 예외를 처리 할 Exception을 구현해준다면 Controller 메소드마다 예외처리를 하지 않아도 된다.

## TO-BE
``` java
@Slf4j
@RestControllerAdvice
@RequiredArgsConstructor
public class ApiControllerAdvice {
    /**
     * 임의로 발생시킨 CustomException처리
     */
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    @ExceptionHandler(LuckKidsException.class)
    public ApiResponse<Object> LuckKidsException(LuckKidsException e) {
        log.error(e.getMessage());
        return ApiResponse.of(
                HttpStatus.BAD_REQUEST,
                e.getMessage(),
                null
        );
    }
}

@RestController
@RequiredArgsConstructor
@RequestMapping("/api/v1/auth")
public class LoginController {

	private final LoginService loginService;

	@PostMapping("/login")
	public ApiResponse<LoginResponse> login(@Valid @RequestBody LoginRequest loginRequest) throws JsonProcessingException {
		return ApiResponse.ok(loginService.normalLogin(loginRequest.toServiceRequest()));
	}

	@PostMapping("/oauth/login")
	public ApiResponse<OAuthLoginResponse> oauthLogin(@Valid @RequestBody LoginOauthRequest loginOauthRequest) throws JsonProcessingException {
		return ApiResponse.ok(loginService.oauthLogin(loginOauthRequest.toServiceRequest()));
	}
}
```