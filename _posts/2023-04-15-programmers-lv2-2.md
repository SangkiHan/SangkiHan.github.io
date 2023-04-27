---
title: 프로그래머스 연속된부분수열의합 Lv2
author: SangkiHan
date: 2023-04-15 16:50:00 +0900
categories: [Java, Algorithm]
tags: [Algorithm]
---
------------

연속된 인덱스들의 합이 K와 같으면 되기에  
합이 K보다 크면 마지막인덱스를 뒤로 보내고  
합이 K보다 작다면 시작인덱스를 뒤로 보내는 식으로 풀이했다.

``` java
public class 연속된부분수열의합 {
	
	public static void main(String[] args) {
		int[] sequence = {1,1,1,2,3,4,5};
		int k = 7;
		solution(sequence, k);
	}
	
	static int[] solution(int[] sequence, int k) {
		int[] answer = new int[2];
		
		int startIndex=0;	//시작 Index값을 담는다
		int endIndex=0;		//마지막 Index값을 담는다.
		int sum=sequence[startIndex];	//Index사이 값들의 합, list첫 값으로 초기화
		int min=Integer.MAX_VALUE;	//제일 작은 값을 넣어줘야하기에 Integer최댓값으로 초기화
		
		//마지막 Index가 요청값 list 길이를 넘어버리면 반복문을 종료한다.
		while(endIndex<sequence.length) {
			//합계가 K와 같고, 두 Index사이 거리가 최솟값보다 작다면 answer에 넣어준다.
			if(sum==k && (endIndex-startIndex)<min) {
				answer[0] = startIndex;
				answer[1] = endIndex;
				min = endIndex-startIndex;
			}
			//합계가 K보다 크다면 시작Index를 앞으로 땡기고 합계해서 빼준다.
			if(sum>k) {
				sum-=sequence[startIndex];
				startIndex++;
			}
			else { //합계가 K보다 작다면 마지막Index를 앞으로 땡기고 합에 더해준다.
				endIndex++;
				if(endIndex!=sequence.length) {
					sum+=sequence[endIndex];
				}
			}
		}
		return answer;
    }
}
```