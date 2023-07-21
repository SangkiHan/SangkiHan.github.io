---
title: Ubuntu 리눅스서버에 MySql설치
author: SangkiHan
date: 2023-07-21 10:44:00 +0900
categories: [Linux, Ubuntu]
tags: [Ubuntu]
---
------------

# MySql 설치
``` bash
sudo apt-get install mysql-server
```
![Jenkins](/assets/img/post/2023-07-21-Mysql-Install/1.PNG)
![Jenkins](/assets/img/post/2023-07-21-Mysql-Install/2.PNG)

# 방화벽 열어주기
MySql은 3306포트로 작동하기 때문에 3306포트를 열어준다.
``` bash
sudo ufw 3306/tcp
```
![Jenkins](/assets/img/post/2023-07-21-Mysql-Install/3.PNG)

# 외부 접근 가능한 IP할당해주기
기본적으로 127.0.0.1 IP만 접근가능하게 MySql에서 기본설정 되어 있다.  
외부에서 접근 가능하도록 IP를 할당해주자.

권한부여
``` bash
CREATE USER 'root'@'%' IDENTIFIED BY '지정할 PASSWORD'; //root계정 갱신
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' WITH GRANT OPTION; //권한부여
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%'; //모든 IP 권한 부여
```

특정IP 권한부여
``` bash
CREATE USER 'root'@'부여할 IP' IDENTIFIED BY '패스워드';
GRANT ALL PRIVILEGES ON *.* TO 'user'@'부여할 IP';
FLUSH PRIVILEGES;
```

권한제거
``` bash
REVOKE ALL PRIVILEGES ON *.* FROM 'root'@'%';
DROP USER root'@'부여된 IP'
```

# DBeaver로 연결
![Jenkins](/assets/img/post/2023-07-21-Mysql-Install/5.PNG)