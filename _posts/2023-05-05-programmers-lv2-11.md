---
title: 프로그래머스 테이블해쉬함수 Lv2
author: SangkiHan
date: 2023-05-05 13:40:00 +0900
categories: [Java, Algorithm]
tags: [Algorithm]
---
------------

Comparator클래스를 사용하여 정렬에 조건을 추가 정렬을 하였다.  
이후 로직은 이중반복문으로 간단하게 구현했다.

``` java
import java.util.Arrays;
import java.util.Comparator;

public class 테이블해쉬함수 {

	public static void main(String[] args) {
		
		int[][] data = new int[][];
		int col = 2;
		int row_begin = 2;
		int row_end = 3;
		solution(data, col, row_begin, row_end);
		
	}
	static  int solution(int[][] data, int col, int row_begin, int row_end) {
        int answer = 0;
        
        Arrays.sort(data, new Comparator<int[]>() {
			@Override
			public int compare(int[] o1, int[] o2) {
				if(o1[col-1]==o2[col-1]) {//col번째 행이 같으면 첫번째행 내림차순
					return o2[0]-o1[0];
				}
				else {
					return o1[col-1]-o2[col-1];//col번째 행이 같지 않으면 col번째행 오름차순
				}
			}
		});
        
        for(int i=row_begin-1; i<row_end; i++) {
        	int num = 0;
        	for(int j=0; j<data[0].length; j++) {
        		num+=data[i][j]%(i+1);//행으로 나누어 나머지 누적
        	}
        	
        	answer^=num;//비트연산자로 num만큼 이동
        }
        
        return answer;
    }
}
```