---
title: 프로그래머스 숫자 변환하기 Lv2
author: SangkiHan
date: 2023-04-24 11:45:00 +0900
categories: [Java, Algorithm]
tags: [Algorithm]
---
------------

처음에는 DFS로 풀이 하였으나 시간초과로 인해 DP로 풀이 하였다.

``` java
public class 숫자변환하기 {
	
	static int min = Integer.MAX_VALUE;
	
	public static void main(String[] args) {
		
		int x = 10;
		int y = 40;
		int n = 5;
		
		solution(x,y,n);

	}
	
	static int solution(int x, int y, int n) {
		int[] array = new int[y+1];
		
		for(int i=x+1; i<array.length; i++) {
			int a = Integer.MAX_VALUE;//최대값으로 세팅
			int b = Integer.MAX_VALUE;
			int c = Integer.MAX_VALUE;
			int d = 0;//*2 *3 -n중 제일 최소인 값을 넣어줄 변수
			if(check(i,2) && (i/2)>=x) {//자연수이고 2로 나누었을 때 x보다 큰 값
				a = array[i/2];
			}
			if(check(i,3) && (i/3)>=x) {//자연수이고 3로 나누었을 때 x보다 큰 값
				b = array[i/3];
			}
			if((i-n)>=x) {//n으로 뺐을 때 x보다 큰 값
				c = array[i-n];
			}
			
			d = Math.min(a, b);//제일 최소값 찾기
			d = Math.min(d, c);
			
			if(d<Integer.MAX_VALUE) {//d가 최대값보다 작을 시 해당인덱스 횟수 +1
				array[i] = d+1;
			}
			else {//최대값일시 최대값 세팅
				array[i] = d;
			}
		}
		
		return (array[y]==Integer.MAX_VALUE)?-1:array[y];//y인덱스의 값이 최대값일시 변환 할 수 없다는 뜻이므로 -1 세팅
	}
	
	static boolean check(int num, int i) {
		boolean check = false;
		
		if(num%i==0 && num/i>0) {
			check = true;
		}
		
		return check;
	}
}
```