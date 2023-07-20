---
title: 프로그래머스 배달 Lv2
author: SangkiHan
date: 2023-07-20 10:44:00 +0900
categories: [Java, Algorithm]
tags: [Algorithm]
---
------------

처음에는 각 지점의 소요시간을 구해야해서 BFS로 풀어야 하나 생각을 했지만 특정지점일 경우만 BFS 각 지점의 모든 소요시간을 구해야하는 경우는 다익스트라를 사용해야해서 다익스트라 알고리즘으로 구현하였다.

``` java
public class 배달 {
	
	public static void main(String[] args) {
		int N=6;
		int K=4;
		
		int[][] road = 	new int[N];
		
		solution(N, road, K);
		
		return;
	}
	
	static int solution(int N, int[][] road, int K) {
		int answer = 0;
		
		int[][] arr = new int[N][N];
		
		/**
		 * 이중배열로 각 index별로 갈 거리 구함
		 * */
		for(int i=0; i<road.length; i++) {
			int start = road[i][0];
			int end = road[i][1];
			int length = road[i][2];
			//똑같은 길이지만 소요시간이 다른경우가 있어 기존에 들어있던 값이 작으면 값을 넣지 않는다.
			if(arr[start-1][end-1]!=0&&arr[start-1][end-1]<length) { 
				continue;
			}
			arr[start-1][end-1] = length;
			arr[end-1][start-1] = length;
		}
		
		//1에서 다른 지점까지 소요되는 시간을 담는 배열
		int[] lengthArr = new int[N];
		//각 지점을 방문처리를 하는 배열
		boolean[] visited = new boolean[N];
		
		//각 배열에 최댓값으로 초기화
		for(int i=1; i<N; i++) {
			lengthArr[i] = Integer.MAX_VALUE;
		}
		
		for(int i=0; i<N; i++) {
			int min=Integer.MAX_VALUE;
			int visitindex = 0;
			for(int j=0; j<N; j++) {
				if(min>lengthArr[j]&&!visited[j]) {//방문하지 않았고, 소요시간 배열중 제일 적게걸린 시간의 인덱스를 가져옴
					min=lengthArr[j];
					visitindex = j;
				}
			}
			for(int j=0; j<N; j++) {
				if(arr[visitindex][j]!=0) {
					/**
					 * 이미 저장되어있는 소요시간 + 현재지점에서 걸리는 소요시간을 더한값이 
					 * 이미 저장되어있는 소요시간보다 작으면 갱신해준다.
					 */
					if(lengthArr[visitindex]+arr[visitindex][j] < lengthArr[j]) { 
						lengthArr[j]=lengthArr[visitindex]+arr[visitindex][j];
					}
				}
				visited[visitindex]=true;//방문처리
			}
		}
		
		/**
		 * K보다 적은 소요시간을 가진 지점을 Count
		 * */
		for(int i=0; i<N; i++) {
			if(lengthArr[i]<=K) {
				answer++;
			}
		}
		
		return answer;
	}
}
```