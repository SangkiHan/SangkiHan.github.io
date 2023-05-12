---
title: 프로그래머스 귤고르기 Lv2
author: SangkiHan
date: 2023-04-30 15:50:00 +0900
categories: [Java, Algorithm]
tags: [Algorithm]
---
------------

처음에는 귤을 List에 담고 특정값이 List에 포함된 개수가 많은 순대로 정렬하도록 하였다.  
하지만 Collections.frequency함수를 많이 쓰는바람에 시간초과가 발생한 것으로 보였다.  

### 오답(시간초과)
``` java
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public class 귤고르기 {

	public static void main(String[] args) {
		int[] tangerine = new int[];
		solution(4, tangerine);
	}
	
	static int solution(int k, int[] tangerine) {
		int answer = 0;
		List<Integer> tangerineList = new ArrayList<>();
		List<Integer> answerList = new ArrayList<>();
		
		for(int a : tangerine) {	//List로 변환
			tangerineList.add(a);
		}
		Collections.sort(tangerineList,(a, b)->{//List를 정렬하면서 List에 포함개수가 클수록 앞에 오도록 정렬
			if(Collections.frequency(tangerineList,a)>Collections.frequency(tangerineList,b)) {
				return -1;
			}
			else if(Collections.frequency(tangerineList,a)==Collections.frequency(tangerineList,b)) {
				return 0;
			}
			else {
				return 1;
			}
		});
		
		for(int i=0; i<k; i++) {//List에 개수만큼 answerList에 넣어줌
			if(!answerList.contains(tangerineList.get(i))) {
				answerList.add(tangerineList.get(i));
			}
		}
		
		answer = answerList.size();
		
		return answer;
	}

}
```

RunTime에러가 발생함으로써 Hash에 귤의 개수를 저장하여 구현하였다.

### 정답
``` java
public class 귤고르기 {

	public static void main(String[] args) {
		int[] tangerine = new int[];
		solution(4, tangerine);
	}
	
	static int solution(int k, int[] tangerine) {
		int answer = 0;
		List<Integer> answerList = new ArrayList<>();
		HashMap<Integer, Integer> countMap = new HashMap<>();
		
		for(int a : tangerine) {
			countMap.put(a, countMap.getOrDefault(a, 0)+1);
		}

		List<Integer> tangerineList = new ArrayList<>(countMap.keySet());
		
		Collections.sort(tangerineList,(a, b)->{
			if(countMap.get(a)>countMap.get(b)) {
				return -1;
			}
			else if(countMap.get(a)==countMap.get(b)) {
				return 0;
			}
			else {
				return 1;
			}
		});
		
		int count=0;
		int index=0;
		while(count<k) {
			count+=countMap.get(tangerineList.get(index));
			answerList.add(tangerineList.get(index));
			index++;
		}
		
		answer = answerList.size();
		
		return answer;
	}

}
```