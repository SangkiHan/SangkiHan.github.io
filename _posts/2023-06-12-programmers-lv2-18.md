---
title: 프로그래머스 방문길이 Lv2
author: SangkiHan
date: 2023-06-12 13:39:00 +0900
categories: [Java, Algorithm]
tags: [Algorithm]
---
------------

방향 String을 Queue에 담아 하나씩 poll하여 출발지와 목적지를 roads(방문길)에 담아서 길들을 지날 떄마다 방문횟수를 +1해주어 구현하였다.

```java
import java.util.ArrayList;
import java.util.LinkedList;
import java.util.List;
import java.util.Queue;

public class 방문길이 {
	
	static int answer = 0;
	static List<String> roads = new ArrayList<>();//지나간 길들을 담는 리스트
	public static void main(String[] args) {
		String dirs = "LULLLLLLU";
		solution(dirs);
	}
	
	static int solution(String dirs) {
		int[] position = new int[2];//목적지를 담을 배열
		Queue<Character> dir = new LinkedList<Character>();//방향을 char로 쪼개서 담을 큐
		for(int i=0; i<dirs.length(); i++) {//쪼개서 담음
			dir.add(dirs.charAt(i));
		}
		position[0]=5;//처음시작지점
		position[1]=5;//처음시작지점
		for(int i=0; i<dirs.length(); i++) {
			int a = 0;
			String road = "";
			road = String.valueOf(position[0])+String.valueOf(position[1]);//출발지저마을 담아둔다
			char move = dir.poll();//큐에서 방향을 가져온다.
			if(move=='U') {//UP일시
				a = position[0]-1;
				if(!check(a)) {
					continue;
				}
				position[0]=a;
			}
			else if(move=='D') {//DOWN일시
				a = position[0]+1;
				if(!check(a)) {
					continue;
				}
				position[0]=a;
			}
			else if(move=='L') {//LEFT일시
				a = position[1]-1;
				if(!check(a)) {
					continue;
				}
				position[1]=a;
			}
			else if(move=='R') {//RIGHT일시
				a = position[1]+1;
				if(!check(a)) {
					continue;
				}
				position[1]=a;
			}
			String destination = String.valueOf(position[0])+String.valueOf(position[1]);//목적지세팅
			if(roadCheck(road, destination)){//해당길을 지나간 경우가 없을시 answer++, roads에 방문한 길 add
				answer++;
				roads.add(road+destination);
			}
		}
		
		return answer;
	}
	
	/*
	 * 범위를 넘어가는 경우 체크
	 * */
	static boolean check(int num) {
		boolean Yn = true;
		if(num<0||num>10) {
			Yn = false;
		}
		return Yn;
	}
	
	/*
	 * 이미 지나간 길인지 체크
	 * */
	static boolean roadCheck(String road, String destination) {
		boolean Yn = true;
		if(roads.contains(road+destination)||roads.contains(destination+road)) {
			Yn = false;
		}
		return Yn;
	}
}
```