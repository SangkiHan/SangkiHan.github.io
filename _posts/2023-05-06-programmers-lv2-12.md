---
title: 프로그래머스 연속 부분 수열 합의 개수 Lv2
author: SangkiHan
date: 2023-05-06 13:40:00 +0900
categories: [Java, Algorithm]
tags: [Algorithm]
---
------------

첫 풀이는 List의 contains함수를 사용하여 중복을 구별하여 구현하였다.  
하지만 contains함수를 남발 하다 보니 시간 초과가 발생하였다.  
그래서 LinkedHashSet를 사용하여 add할 때 바로 중복제거를 할 수 있도록 변경하여 구현하였다.

### 오답(시간초과)
``` java
import java.util.*; 
class Solution {
    public int solution(int[] elements) {
        int answer = 0;
        List<Integer> list = new ArrayList<>();
        List<Integer> answerList = new ArrayList<>();
        
        for(int j=0; j<2; j++) {
        	for(int i=0; i<elements.length; i++) {
            	list.add(elements[i]);
            }
        }
        
        for(int i=0; i<elements.length; i++) {
        	for(int j=0; j<elements.length; j++) {
        		int num = 0;
        		for(int z=0; z<i+1; z++) {
        			num+=list.get(j+z);
        		}
        		if(!answerList.contains(num)) {
            		answerList.add(num);
            	}
        	}
        }
        
        answer = answerList.size();
        
        return answer;
    }
}
```

### 정답
``` java
import java.util.*; 
class Solution {
    public int solution(int[] elements) {
        int answer = 0;
        List<Integer> list = new ArrayList<>(); //elements에서 더해야하는 길이가 elements를 초과하면 첫번째 인덱스로 이동하기 때문에 2배로 길이를 늘렸다
        Set<Integer> answerSet = new LinkedHashSet<Integer>();
        
        for(int j=0; j<2; j++) {//elements 길이 2배로 늘림
        	for(int i=0; i<elements.length; i++) {
            	list.add(elements[i]);
            }
        }
        
        for(int i=0; i<elements.length; i++) {
        	for(int j=0; j<elements.length; j++) {
        		int num = 0;//더한 합을 담음
        		for(int z=0; z<i+1; z++) {
        			num+=list.get(j+z);
        		}
        		answerSet.add(num);
        	}
        }
        
        answer = answerSet.size();
        
        return answer;
    }
}
```