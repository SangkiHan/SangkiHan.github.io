---
title: 프로그래머스 두 큐의 합 같게 만들기 Lv2
author: SangkiHan
date: 2023-05-01 13:50:00 +0900
categories: [Java, Algorithm]
tags: [Algorithm]
---
------------

두 큐의 합을 비교하여 작은 큐의 값을 poll하여 큰 큐로 add 하도록 구현하였다.  
두 큐가 도저히 같아지지 않는 조건을 찾기 못하여 answer가 300000회를 넘으면 -1을 return 하도록 구현하였다...

``` java
import java.util.LinkedList;
import java.util.Queue;

public class 두큐의합같게만들기 {
    
	public static void main(String[] args) {
		
		int[] queue1 = {1, 2, 1, 2};
		int[] queue2 = {1, 10, 1, 2};
		
		solution(queue1, queue2);

	}
	
	static int solution(int[] queue1, int[] queue2) {
		int answer = 0;
		
		Queue<Long> queueOne = new LinkedList<>();//첫번째 큐
	    Queue<Long> queueTwo = new LinkedList<>();//두번째 큐
	    long sum = 0;//두큐의 총합
	    long oneSum = 0;//첫번째 큐의 합
	    long twoSum = 0;//두번째 큐의 합
	    
        for(int num : queue1){//첫번째 큐 초기화
            queueOne.add((long) num);
            sum+=num;
            oneSum+=num;
        }
        
        for(int num : queue2){//두번째 큐 초기화
        	queueTwo.add((long) num);
            sum+=num;
            twoSum+=num;
        }
        
        if(sum%2>0) {//총합의 나머지가 있으면 -1
        	answer=-1;
        }
        else {
        	while(oneSum!=twoSum) {//두큐의 합이 같아 질 때까지 반복
        		if(oneSum>twoSum) {//첫번째 큐 > 두번째 큐
        			long num = queueOne.poll();
        			queueTwo.add(queueOne.poll());
        			oneSum -= num;
        			twoSum += num;
        		}
        		else {//두번째 큐 > 첫번째 큐
        			long num = queueTwo.poll();
        			queueOne.add(num);
        			oneSum += num;
        			twoSum -= num;
        		}
        		if(answer>300000){//두큐가 도저히 같아지지 않을 때의 경우를 찾아야하는데 찾지못하여 큐의 최대길이인 300000회를 넘으면 같아지지않는다고 판단했다...
                    answer=-1;
                    break;
                }
        		answer++;
        	}
        }
        
        return answer;
    }
}
```