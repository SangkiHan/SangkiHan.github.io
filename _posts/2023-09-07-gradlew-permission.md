---
title: Gradle Permission
author: SangkiHan
date: 2023-09-06 11:20:00 +0900
categories: [CI/CD, Jenkins]
tags: [CI/CD]
---
------------

``` bash
0.213 /bin/sh: ./gradlew: Permission denied
------
Dockerfile:9
--------------------
   7 |     COPY . /home/project/FileTransfer
   8 |     
   9 | >>> RUN ./gradlew clean
  10 |     RUN ./gradlew bootWar
  11 |     
--------------------
 ```

 ``` bash
git ls-tree HEAD
git update-index --add --chmod=+x gradlew
git add .
git commit -m "fix: permission"
git push
 ```