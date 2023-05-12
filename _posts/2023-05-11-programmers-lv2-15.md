---
title: 프로그래머스 혼자놀기의 달인 Lv2
author: SangkiHan
date: 2023-05-11 16:50:00 +0900
categories: [Java, Algorithm]
tags: [Algorithm]
---
------------

Hash에 상자카드번호 index를 같이 담아 그룹을 만드는 식으로 간단히 풀이했다.

``` java
import java.util.ArrayList;
import java.util.Collections;
import java.util.HashMap;
import java.util.List;

public class 혼자놀기의달인 {
	
	static int answer = 0;
	
	public static void main(String[] args) {
		int[] card = { 2, 3, 4, 5, 6, 7, 8, 9, 10 , 1};
		solution(card); 
	}
	
	static int solution(int[] cards) {
		
		HashMap<Integer, Integer> indexHash = new HashMap<Integer, Integer>();
		List<Integer> answerList = new ArrayList<Integer>();
		
		for(int i=0; i<cards.length; i++) {//카드별 상자번호 저장
			indexHash.put(cards[i], i);
		}
		
		int totalNum = 0;
		int index = 1;//첫상자
		
		while(totalNum!=cards.length) {//전체 상자를 다 돌았으면 break;
			int boxNum = 0;
			int box = indexHash.get(index);
			while(cards[box]!=0) {//카드상자가 0이면 break;
				int card = cards[box];
				cards[box]=0;
				box = card-1;
				boxNum++;
			}
			index++;
			totalNum+=boxNum;
			answerList.add(boxNum);
		}
		if(answerList.size()>1) {//담겨있는 상자개수가 2개이상이면 첫번째 두번째상자 곱해줌
			Collections.sort(answerList, Collections.reverseOrder());
			answer = answerList.get(0) * answerList.get(1);
		}
		else {//담겨있는 상자개수가 한개면 두번째상자가 없으므로 0
			answer = 0;
		}
        return answer;
    }
}
```