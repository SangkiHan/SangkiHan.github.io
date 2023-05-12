---
title: 프로그래머스 피로도 Lv2
author: SangkiHan
date: 2023-05-12 10:50:00 +0900
categories: [Java, Algorithm]
tags: [Algorithm]
---
------------

문제유형에 완전탐색이라는 힌트가 있어 방문배열을 생성하여 재귀함수로 완전탐색하여 구현하였다.

``` java
public class 피로도 {
	static int answer = 0;
	public static void main(String[] args) {
		int[][] dungeons = new int[][];
		int k = 80;
		solution(k,dungeons);
	}
	
	static int solution(int k, int[][] dungeons) {
		boolean[] visited = new boolean[dungeons.length];//방문한 위치
        
		dfs(0, k, dungeons, visited.clone());
        
        return answer;
    }
	
	static void dfs(int num, int k, int[][] dungeons, boolean[] visited) {
		for(int i=0; i<dungeons.length; i++) {
			int k2 = dungeons[i][0];//해당 인덱스 배열에서 제한 피로도 가져옴
			int vit = dungeons[i][1];//해당 인덱스 배열에서 소모 피로도 가져옴
			if(k<k2 || k<vit || visited[i]) {//제한피로도에 못 미치거나 소모 피로도만큼 피로도가 있지 않던가 방문했다면 재귀 끝
				continue;
			}
			visited[i] = true;//방문처리 
			num++;//던전방문횟수 증가
			if(answer<num) {//정답보다 크면 바꿔줌
				answer=num;
			}
			dfs(num, k-vit, dungeons, visited.clone());//재귀다시돌림
			visited[i] = false;//첫번째 던전에 방문하지 않은걸로 바꿔둠
			num--;//첫번째 던전에 방문하지 않은 것으로 바꿨기 때문에 방문횟수도 -1
		}
	}
}
```