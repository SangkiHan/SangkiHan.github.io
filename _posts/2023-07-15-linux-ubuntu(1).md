---
title: 개인 노트북으로 우분투 서버 설치(1)
author: SangkiHan
date: 2023-07-15 10:44:00 +0900
categories: [Linux, Ubuntu]
tags: [Ubuntu]
---
------------

리눅스 서버에 구축 실습을 할 때 항상 AWS프리티어를 이용하여 실습을 하였다.
하지만 일정 시간이 지날 시 요금을 내게 되어있어 너무 불편하고 요금이 초과되지 않을까 항상 고민됐었다.
그래서 집에 남는 노트북을 이용하여 우분트를 설치하고 개인서버로 활용하기로 생각했다.

------------
## 우분투 ISO다운

https://ubuntu.com/download/desktop

필자는 제일 최신 버전인 Ubuntu 22.04.2 LTS를 사용하였다.

![Ubuntu](/assets/img/post/2023-07-15-linux-ubuntu(1)/1.PNG)

------------
## ISO파일을 담을 부팅USB 만들기
ISO파일 담으려면 적어도 4GB이상의 USB가 필요하다.
다이소에서 16GB USB를 7500원에 구입하였다.

아래 URL로 rufus라는 부팅USB를 만들어주는 프로그램을 다운받자  
https://rufus.ie/ko/

![Ubuntu](/assets/img/post/2023-07-15-linux-ubuntu(1)/2.PNG){: width="50%"}

장치를 선택 후 부팅할 ISO를 선택하고 시작을 하면된다.
필자는 5분정도 소요되었다.

------------
## 개인노트북 설치
부팅USB를 설치할 노트북에 꽂고 BIOS설정 화면으로 들어가자
필자는 ASUS사의 제품이라 부팅시 esc를 누르면 BIOS설청창으로 들어가졌지만 제조자마다 다르니 검색해보자

부팅할 USB를 선택하고 아래와 같은 우분투 화면이 뜨면 성공이다.
![Ubuntu](/assets/img/post/2023-07-15-linux-ubuntu(1)/3.PNG){: width="50%"}

한국어 선택 후 Ubuntu를 설치하면된다.
![Ubuntu](/assets/img/post/2023-07-15-linux-ubuntu(1)/4.PNG){: width="50%"}

필자는 이제 개인노트북에 윈도우가 필요하지 않아 설정에서 없애주었다.
![Ubuntu](/assets/img/post/2023-07-15-linux-ubuntu(1)/5.PNG){: width="50%"}

설치가 완료되고 아래와 같은 배경이 뜨면 설치가 완료된다.
![Ubuntu](/assets/img/post/2023-07-15-linux-ubuntu(1)/6.PNG){: width="50%"}