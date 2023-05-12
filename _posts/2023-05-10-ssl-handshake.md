---
title: SSL 핸드쉐이크란?
author: SangkiHan
date: 2023-05-10 16:50:00 +0900
categories: [Http]
tags: [Http]
---
------------

![HandShake](/assets/img/post/2023-05-02-ssl-handshake/handshake.png)

## 과정(https://~ 접속 이후)
1.  Client -> Server
    -   SSL버전 정보
    -   암호화 방식
    -   무작위 바이트 문자열을 만듬
2.  Server -> Client
    -   서버가 사용하는 암호화 방식
    -   무작위 바이트 문자열을 만듬
    -   SESSIONID
    -   SSL인증서
3.  Client
    -   서버가 보내준 인증서를 CA를 통해 확인
    -   CA를 통해 발급된 정상적인 인증서인지 확인한다.
        +   정상적으로 발급 된 인증서가 아닐 경우 브라우저에 경고표시가 뜨게 된다.
4.  Client
    -   1과 2과정에서 Client와 Server가 주고 받은 무작위 바이트 문자열을 조합하여 임시 key를 만든다.
5.  Client
    -   임시 Key를 인증서의 공개키로 암호화 한다.
6.  Client -> Server
    -   암호화 된 임시 Key를 서버로 보내준다.
7.  Server
    -   Client에서 받은 임시 Key를 서버에서도 공개키로 복호화가 가능하다. 이로써 서버와 클라이언트가 임시Key 공유가 가능해졌다.

#### 위 과정을 거치고 만들어진 임시key로 Client와 Server가 데이터를 암호화 하여 주고받을 수 있다.