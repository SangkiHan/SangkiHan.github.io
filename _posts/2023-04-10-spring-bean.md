---
title: Spring Bean
author: SangkiHan
date: 2023-04-10 12:32:00 +0900
categories: [Spring]
tags: [Spring]
---

## Bean이란? 
Spring IoC 컨테이너가 관리하는 자바 객체를 빈(bean)이라고 한다.

------------
### Bean사용이유
1.  new를 사용하여 객체를 생성하지 않고 Spring IOC에서 미리 생성된 Bean을 가져다 쓰기 위해 사용한다. 
2.  객체를 IOC컨테이너가 관리하게 되어 객체를 효율적으로 관리가 가능하다. Bean으로 등록된 동일한 객체를 가져다 사용하기 때문에 메모리를 적게 사용하는 장점이 있다.
3.  객체가 미리 생성되기 이전에 검사가 가능하다.

------------
### Bean등록방법
1.  component-scan기능을 사용하여 Bean등록을 할 패키지 범위를 설정해준다. 
    ``` xml
        <?xml version="1.0" encoding="UTF-8"?>
        <beans xmlns="http://www.springframework.org/schema/beans"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xmlns:context="http://www.springframework.org/schema/context"
            xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context https://www.springframework.org/schema/context/spring-context.xsd">

            <context:component-scan base-package="com.spring.study" />


        </beans>
    ```

        패키지 범위 안 각 클래스위에 어노테이션을 붙혀준다.

    +   Contoller    
    ``` java
        import org.springframework.stereotype.Controller;

        @Controller
        public class TestContoller {
        }   
    ```

    +   Service
    ``` java
        import org.springframework.stereotype.Service;

        @Service
        public class TestService {
        } 
    ```

    +   Repository
    ``` java
        import org.springframework.stereotype.Repository;

        @Repository
        public class TestRepository {
        }
    ```

    +   Config
    ``` java
        import org.springframework.context.annotation.Configuration;

        @Configuration
        public class SecurityConfig{
        }
    ```

2.  XML에 Bean 수동등록
    ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

        <bean id="TestService"
            class="com.spring.study.TestService">
            <property name="TestRepository" ref="testRepository" />
        </bean>

        <bean id="testRepository"
            class="com.spring.study.testRepository" />
    </beans>
    ```


