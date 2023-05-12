---
title: 프로그래머스 할인행사 Lv2
author: SangkiHan
date: 2023-05-03 11:45:00 +0900
categories: [Java, Algorithm]
tags: [Algorithm]
---
------------

HashMap을 생성하여 할인상품들의 개수를 key value로 담아주고  
Queu를 생성하여 10일간의 할인상품들을 담아줬다.

``` java
import java.util.HashMap;
import java.util.LinkedList;
import java.util.Queue;

public class 할인행사 {

	public static void main(String[] args) {
		
		String[] want = {"banana", "apple", "rice", "pork", "pot"};
		int[] number = {3, 2, 2, 2, 1};
		String[] discount = {"chicken", "apple", "apple", "banana", "rice", "apple", "pork", "banana", "pork", "rice", "pot", "banana", "apple", "banana"};
		solution(want, number, discount);
	}
	
	static int solution(String[] want, int[] number, String[] discount) {
        int answer = 0;
        Queue<String> discountList = new LinkedList<>();//할인 상품을 담을 Queue
        HashMap<String, Integer> discountNum = new HashMap<>();//10일간의 할인상품을 담을 해쉬
        int index;
        for(index=0; index<10; index++) {//첫날부터 10일까지의 상품으로 초기화
        	discountList.add(discount[index]);
        	discountNum.put(discount[index],discountNum.getOrDefault(discount[index], 0)+1);
        }
        
        while(index!=discount.length+1) {//index가 discount의 최대값까지 도달하면 끝
        	int i;
        	for(i=0; i<want.length; i++) {
        		if(discountNum.getOrDefault(want[i],0)!=number[i]) {//want상품이 10일간의 할인 개수에 포함이 안될시 break;
        			break;
        		}
        	}
        	
        	if(i==want.length) {//break되지않고 할인상품이 전부 있다면 answer++
        		answer++;
        	}
        	
        	if(index!=discount.length) {//index가 최대값에 도달하지않았다면 queue에서 첫값 빼와서 해쉬에서 -1, 다음날 할인상품 +1
            	String first = discountList.poll();
            	String last = discount[index];
            	discountNum.put(first, discountNum.getOrDefault(first,0)-1);
            	discountNum.put(last, discountNum.getOrDefault(last,0)+1);
            	discountList.add(last);
        	}
        	index++;
        }
        return answer;
    }

}
```