---
title: Functional Interface
author: SangkiHan
date: 2023-05-01 13:50:00 +0900
categories: [Java]
tags: [Java]
---

#### ````Functional Interface````란 Java8부터 도입된 람다식을 이용한 함수형 프로그래밍을 가능하게 하는 인터페이스이다.

1.  Consumer
    -   파라미터를 받아서 아무런 값을 반환하지 않는 Functional Interface이다.
    -   파라미터의 타입을 T로 받아 유동적으로 사용가능하다.
    -   Interface
        ``` java
        @FunctionalInterface
        public interface Consumer<T> {
            void accept(T t);
        }
        ```
    -   사용예시
        ``` java
        //List를 foreach로 출력할 때 사용할 수 있다.
        Consumer<String> consumer = s -> System.out.println(s);
        List<String> list = Arrays.asList("A", "B", "C");
        list.forEach(consumer);
        ```

2.  Supplier
    -   아무 파라미터를 받지 않고 응답값만 반환하는 Functional Interface이다.
    -   반환 할 응답값의 타입을 T로 지정한다.
    -   Interface
        ``` java
        @FunctionalInterface
        public interface Supplier<T> {
            T get();
        }
        ```
    -   사용예시
        ``` java
        Supplier<String> supplier = () -> "hello world";
        String result = supplier.get();
        System.out.println(result);
        ```

3.  Fucntion
    -   파라미터를 받아서 응답값을 반환하는 Functional Interface이다.
    -   파라미터 타입을 T로, 반환 할 응답값의 타입을 R로 지정한다.
    -   Interface
        ``` java
        @FunctionalInterface
        public interface Function<T, R> {
            R apply(T t);
        }
        ```
    -   사용예시
        ``` java
        //문자열을 숫자로 변환하는 등의 작업에 사용할 수 있다.
        Function<String, Integer> function = s -> Integer.parseInt(s);
        int result = function.apply("100");
        System.out.println(result);
        ```

4.  Predicate
    -   파라미터를 받아서 응답값을 Boolean으로 반환하는 Functional Interface이다.
    -   파라미터 타입을 T로 지정한다.
    -   Interface
        ``` java
        @FunctionalInterface
        public interface Predicate<T> {
            boolean test(T t);
        }
        ```
    -   사용예시
        ``` java
        //필터링 등에 사용할 수 있다.
        Predicate<Integer> predicate = i -> i > 10;
        boolean result = predicate.test(5);
        System.out.println(result);
        ```

5.  UnaryOperator
    -   파라미터와 응답값의 타입이 모두 T인 Functional Interface이다.
    -   Interface
        ``` java
        @FunctionalInterface
        public interface UnaryOperator<T> extends Function<T, T> {
            static <T> UnaryOperator<T> identity() {
                return t -> t;
            }
        }
        ```
    -   사용예시
        ``` java
        //값을 변환하는 등의 작업에 사용할 수 있습니다.
        UnaryOperator<Integer> operator = i -> i * 2;
        int result = operator.apply(10);
        System.out.println(result);
        ```