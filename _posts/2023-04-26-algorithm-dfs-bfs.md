---
title: DFS(깊이 우선 탐색), BFS(너비 우선 탐색)
author: SangkiHan
date: 2023-04-26 15:50:00 +0900
categories: [Java, Algorithm]
tags: [Algorithm]
---

------------
## 그래프

**정점(node)**과 그 정점을 연결하는 **간선(edge)**으로 이루어진 자료구조의 일종

------------
## 깊이 우선 탐색(DFS)

![DFS](/assets/img/post/2023-04-26-algorithm-dfs-bfs/dfs.PNG)

**정점(node)**에서 시작하여 다음 분기로 넘어가지 전 해당 분기를 전부 탐색하는 방식입니다.

### 특징
-   모든 node를 탐색해야 할 경우 해당 방법 사용
-   BFS보다 사용이 간단함
-   검색 속도가 BFS에 비해서 느림ㄴ

**Stack** 또는 **재귀함수**를 사용하여 구현한다.

------------
## 너비 우선 탐색(BFS)

![DFS](/assets/img/post/2023-04-26-algorithm-dfs-bfs/bfs.PNG)

**정점(node)**에서 시작하여 인접한 node들을 먼저 탐색하는 방식입니다.

### 특징
-   최단경로 같은 완전탐색을 필요로 하지 않을 때 사용

**Queue**를 사용하여 구현한다.