---
title: Spring MVC 동작원리
author: SangkiHan
date: 2023-05-31 09:39:00 +0900
categories: [Spring]
tags: [Spring]
---
------------
## 동작원리
![SpringMVC](/assets/img/post/2023-05-31-spring-mvc/1.PNG)

1. DispatcherServlet이 요청을 받음
2. 요청된 URL을 HandlerMapping에 보내고, 해당 요청의 URL이 Controller에 있는지 확인을 해본다.
    + 없으면 Exception
    + 있으면 Controller의 메소드정보를 DispatcherServlet에 return해준다.
3. DispatcherServlet이 HandlerAdapter를 가져온다.
4. HandlerAdapter는 2번 HandlerMapping으로 찾은 Controller의 메소드를 실행한다. 
    + Controller메소드에 Service(비지니스로직), Repository(DB), DB를 거쳐서 로직을 수행한다. 
5. Controller에서 viewName을 HandlerAdapter에게 return해준다. HandlerAdapter는 DispatcherServlet에게 viewName를 다시 전달해준다.
6. DispatcherServlet은 전달 받은 viewName을 ViewResolver에게 전달하여 view객체를 받는다.
7. DispatcherServlet은 View객체를 화면 표시 의뢰를 한다. 
8. View객체가 해당하는 뷰를 호출한다.

------------

## 구성요소

1. **DispatcherServlet** : Front Controller라고도 불리며 HTTP 프로토콜로 들어오는 모든 요청을 먼저 받아서 적합한 컨트롤러에 위임하는 역활을 함
    + 원래는 web.xml에 모든 Sevlet에 대해 URL를 매핑하기 위해 설정을 해줬어야 하지만 DispatcherServlet의 등장이후 모든 요청을 받아 알아서 Controller로 위임해주기 때문에 개발이 편리해졌다.
    >**TIP**  
    >Servlet이란?  
    >Servlet은 웹 프로그래밍에서 클라이언트의 요청을 처리하고 처리 결과를 클라이언트로 전송하는 기술이다.
2. **HandlerMapping** : HTTP 요청 정보를 확인하여 Controller를 찾아주는 기능을 수행한다.
3. **HandlerAdapter** : Controller에서 처리한 return값을 받아 ModelAndView형태로 바꿔 DispatcherServlet에 전달하는 기능을 수행한다.
4. **ViewResolver** : DispatcherSevlet에서 전달받은 View객체로 실행할 View를 찾는 기능을 수행한다.
