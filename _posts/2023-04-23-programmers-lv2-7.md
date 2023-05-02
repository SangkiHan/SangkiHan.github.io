---
title: 프로그래머스 뒤에 있는 큰 수 찾기 Lv2
author: SangkiHan
date: 2023-04-23 14:45:00 +0900
categories: [Java, Algorithm]
tags: [Algorithm]
---
------------

처음에는 완전탐색으로 풀이했지만 시간초과가 발생하여 완전탐색으로 하는 풀이는 잘못됐다고 생각하였다.

#### 오답(시간초과)
``` java
import java.util.ArrayList;
import java.util.List;

public class 뒤에있는큰수찾기 {

	public static void main(String[] args) {
		int[] numbers = new int[];
		solution(numbers);
	}
	
	static List<Integer> solution(int[] numbers) {
        List<Integer> answer = new ArrayList<>();
        
        for(int i=0; i<numbers.length; i++) {//완전탐색
        	for(int j=i; j<numbers.length; j++) {
        		if(numbers[i]<numbers[j]) {//다음숫자중 큰 수가 나올 때 까지 반복
        			answer.add(numbers[j]);
        			break;
        		}
        		else if(j==numbers.length-1){//큰 수가 발견되지 않을 시에 -1
        			answer.add(-1);
        			break;
        		}
        	}
        }
 
        return answer;
    }
}
```

그러므로 Stack이용하여 안에 해당 numbers보다 큰 수들의 index를 담아두고 stack을 뒤지는 식으로 풀이하였다.

#### 성공
``` java
import java.util.Stack;

public class 뒤에있는큰수찾기 {

	public static void main(String[] args) {
		int[] numbers = new int[];
		solution(numbers);
	}
	
	static int[] solution(int[] numbers) {
        int[] answer = new int[numbers.length];
        Stack<Integer> stack = new Stack<>();
        stack.add(0);//Stack에 초기값 0을 담아둠
        
        for(int i=1; i<numbers.length; i++) {
        	
        	while(!stack.isEmpty() && numbers[stack.peek()]<numbers[i]) {//stack이 비어있지않고 stack에 index의 있는 numbers의 숫자가 더 작을시
        		answer[stack.pop()]=numbers[i];//stack 첫행 삭제후 answer에 넣어줌
        	}
        	stack.add(i);
        }
        
        while(!stack.isEmpty()) {//stack에 아직도 숫자가 남아있다는것은 index다음 큰 숫자가 없었다는 뜻
        	answer[stack.pop()]=-1;
        }
 
        return answer;
    }
}
```