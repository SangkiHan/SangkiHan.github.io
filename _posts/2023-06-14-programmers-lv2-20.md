---
title: 프로그래머스 전력망을 둘로 나누기 Lv2
author: SangkiHan
date: 2023-06-14 10:44:00 +0900
categories: [Java, Algorithm]
tags: [Algorithm]
---
------------

HashMap에 모든 노드들의 연결상태를 저장하고 bfs로 노드들의 연결 상태를 하나하나 끊으며 정답을 구하였다.

``` java
import java.util.HashMap;
import java.util.Iterator;
import java.util.LinkedList;
import java.util.Queue;

public class 전력망을둘로나누기 {
	
	static HashMap<Integer, Queue<Integer>> linked;//각 노드가 연결되어있는 노드를 저장하는 Hash
	static int N=0;//노드의 수
	static int firstStart = 0;//시작 노드
	static int answer = Integer.MAX_VALUE;//정답

	public static void main(String[] args) {
		int[][] wires = {{1,3},{2,3},{3,4},{4,5},{4,6},{4,7},{7,8},{7,9}};
		int n = 9;
		solution(n, wires);
	}
	
	static int solution(int n, int[][] wires) {
        N=n;
        linked = new HashMap<>();
        firstStart = wires[0][0];
        for(int i=0; i<wires.length; i++) {
        	Queue<Integer> startlink;
        	Queue<Integer> endlink;
        	
        	int start = wires[i][0];
        	int end = wires[i][1];
        	
        	startlink = linked.getOrDefault(start, new LinkedList<>());//해당 노드의 연결되어있는 노드를 넣어줌
        	startlink.add(end);
        	linked.put(start, startlink);
        	
        	endlink = linked.getOrDefault(end, new LinkedList<>());//위에서 넣어준 노드의 반대로 넣어줌
        	endlink.add(start);
        	linked.put(end, endlink);
        }
        
        Iterator<Integer> iterator = linked.keySet().iterator();
        
        while(iterator.hasNext()) {//저장한 해쉬의 키값을 반복함
        	int first = iterator.next();
        	Queue<Integer> list = linked.getOrDefault(first, new LinkedList<>());//해당 노드가 연결되어있는 노드들을 가져옴
        	for(int i=0; i<list.size(); i++) {//연결된 노드만큼 반복
        		int linkNum = list.poll();//노드의 연결을 끊음
            	linked.put(first, list);
            	int linkSize = bfs();//bfs돌림
            	list.add(linkNum);//끊은 노드의 연결을 다시 연결해줌
            	linked.put(first, list);
            	int a = Math.abs(linkSize-(N-linkSize));//연결을 끊었을 때 두 묶음의 차이를 절대값으로 바꿔줌
        		if(a<answer) {//answer보다 절대값이 작으면 넣어줌
        			answer=a;
        		}
        	}
        }
        return answer;
    }
	
	static int bfs() {
		Queue<Integer> visit = new LinkedList<Integer>();//방문해야 할 노드들
		int linkSize=1;
		
    	boolean[] visited = new boolean[N];
		visit.add(firstStart);//처음 시작은 임의로 넣어줌
		
		while(!visit.isEmpty()) {//방문해야 할 노드들이 비어있을 때 까지 반복
			int start = visit.poll();//방문해야할 노드를 하나 가져옴
			Queue<Integer> list = linked.getOrDefault(start, new LinkedList<>());//방문해야할 노드가 연결되어있는 노드들을 가져옴
			for(int end : list) {//노드들을 전부 방문
				if(visited[end-1]==false) {//방문하지 않았다면
					visit.add(end);//방문하지 않은 노드 넣어줌
					linkSize++;//연결되어 있는 노드 숫자 +1
					visited[end-1]=true;//방문처리
				}
			}
			visited[start-1]=true;//다른 노드들이 다시 방문하는것을 방지하기 위해 시작노드도 방문처리
		}
		return linkSize;
	}
}
```