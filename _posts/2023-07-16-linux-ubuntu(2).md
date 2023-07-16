---
title: 개인 노트북으로 우분투 서버 원격접속하기(2)
author: SangkiHan
date: 2023-07-16 10:44:00 +0900
categories: [Linux, Ubuntu]
tags: [Ubuntu]
---
------------
## 내부 라이브러리 최신화

```bash
sudo apt update
```

------------
## SSH설치
SSH란 원격지 호스트 컴퓨터에 접속하기 위해 사용되는 인터넷 프로토콜이다.

SSH를 설치해야 원격으로 외부접속이 가능하다.

```bash
sudo apt install net-tools
```

SSH설치가 완료 되었다면 포트를 설정하자
SSH는 기본포트가 22로 되어있어 22그대로 사용할 것이라면 굳이 설정을 안해도 되지만
보안상 설정을 해줘야한다면 아래와 같이 vi로 sshd_config파일을 열어 포트를 바꿔주면된다

![Ubuntu](/assets/img/post/2023-07-16-linux-ubuntu(2)/1.PNG){: width="50%"}

```bash
sudo vi /etx/ssh/sshd_config //설정파일 열기
sudo /etc/init.d/ssh restart //SSH Restart
```

변경포트 확인
```bash
sudo netstat -anp|grep LISTEN|grep sshd
```

포트가 변경되었다면 완료다

------------
## 방화벽 확인
SSH가 설치가 되었다고 해도 방화벽이 설정이 되어있으면 접속이 불가할 수 있다.
ufw명령어로 방화벽 여부를 확인 할 수 있다.
하지만 root 권한 일 경우만 사용이 가능하기 때문에 su로 root권한으로 바꿔주자

```bash
su //root권한으로 변경
//비밀번호 에러 난다면 초기비밀번호가 설정되어있지 않아서 그런거니 아래 명령어로 비밀번호를 설정해주자
sudo passwd //비밀번호 변경
```

root권한으로 변경했다면 방화벽 여부를 확인해보자

아래 명령어로 확인 할 수 있다.
```bash
ufw status //방화벽 확인
```
![Ubuntu](/assets/img/post/2023-07-16-linux-ubuntu(2)/2.PNG){: width="60%"}

비활성화가 되어있다면 원격접속에는 문제가 없다. 
하지만 활성화를 해주고 싶다면 아래와 같이 해주면 된다.

```bash
ufw enable //방화벽 활성화
ufw allow 22/tcp //22번 포트만 열어줌
```
![Ubuntu](/assets/img/post/2023-07-16-linux-ubuntu(2)/3.PNG){: width="60%"}
![Ubuntu](/assets/img/post/2023-07-16-linux-ubuntu(2)/4.PNG){: width="60%"}

------------
## 원격접속

```bash
ssh -Y itricat@ip주소
```
![Ubuntu](/assets/img/post/2023-07-16-linux-ubuntu(2)/5.PNG){: width="60%"}