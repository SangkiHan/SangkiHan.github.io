---
title: Restful Api?
author: SangkiHan
date: 2023-06-08 12:39:00 +0900
categories: [Http]
tags: [Http]
---
------------

## Rest란?

Rest는 자원을 이름으로 구분하여 해당 자원의 상태를 주고받는 것입니다.

즉, HTTP URI를 통해 자원을 명시하고, HTTP Method를 통해 해당 URI에 대한 CRUD를 적용하는 것을 의미않다.

>**HTTP Method**  
>GET: 자원조회  
>POST: 요청 데이터 처리, 주로 등록시 사용  
>PUT: 자원을 대체, 생성시 사용   
>DELETE: 자원 삭제  
>PATCH: 자원을 부분만 대체


>**CRUD**  
>CREATE: 생성  
>READ: 읽기  
>UPDATE: 수정  
>DELETE: 삭제

### Rest의 구성요소
1.  자원: HTTP URI
2.  자원에 대한 행위: HTTP Method
3.  자원에 대한 행위의 내용: HTTP Message Pay Load(요청값 혹은 응답값)

## Rest API란?
REST의 원리를 따르는 API를 의미한다. 

### REST 설계 예시
#### 1. 동사보다는 명사, 대문자보다는 소문자를 사용한다.
```java
틀린예
@RequestMapping(value = "/REST-API" , method = {RequestMethod.GET, RequestMethod.POST})
@ResponseBody
public void restApi(@RequestParam("request") String request) {
    return "response";
}

옳은예
@RequestMapping(value = "/rest-api" , method = {RequestMethod.GET, RequestMethod.POST})
@ResponseBody
public void restApi(@RequestParam("request") String request) {
    return "response";
}
```

#### 2. 마지막에 /을 포함하지 않는다.
```java
틀린예
@RequestMapping(value = "/rest-api/" , method = {RequestMethod.GET, RequestMethod.POST})
@ResponseBody
public void restApi(@RequestParam("request") String request) {
    return "response";
}

옳은예
@RequestMapping(value = "/rest-api" , method = {RequestMethod.GET, RequestMethod.POST})
@ResponseBody
public void restApi(@RequestParam("request") String request) {
    return "response";
}
```

#### 3. _대신 -를 사용한다.
```java
틀린예
@RequestMapping(value = "/rest_api" , method = {RequestMethod.GET, RequestMethod.POST})
@ResponseBody
public void restApi(@RequestParam("request") String request) {
    return "response";
}

옳은예
@RequestMapping(value = "/rest-api" , method = {RequestMethod.GET, RequestMethod.POST})
@ResponseBody
public void restApi(@RequestParam("request") String request) {
    return "response";
}
```

#### 4. 파일관련 API일시 확장자명은 포함하지않는다.
```java
틀린예
@RequestMapping(value = "/rest-api.jpg" , method = {RequestMethod.GET, RequestMethod.POST})
@ResponseBody
public void restApi(@RequestParam("request") String request) {
    return "response";
}

옳은예
@RequestMapping(value = "/rest-api" , method = {RequestMethod.GET, RequestMethod.POST})
@ResponseBody
public void restApi(@RequestParam("request") String request) {
    return "response";
}
```

#### 5. 어떤 HTTP Method인지 포함하지 않는다.
```java
@RequestMapping(value = "/post/rest-api" , method = {RequestMethod.GET, RequestMethod.POST})
@ResponseBody
public void restApi(@RequestParam("request") String request) {
    return "response";
}

옳은예
@RequestMapping(value = "/rest-api" , method = {RequestMethod.GET, RequestMethod.POST})
@ResponseBody
public void restApi(@RequestParam("request") String request) {
    return "response";
}
```
