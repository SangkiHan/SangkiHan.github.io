---
title: 프로그래머스 영어 끝말잇기 Lv2
author: SangkiHan
date: 2023-08-10 16:44:00 +0900
categories: [Java, Algorithm]
tags: [Algorithm]
---
------------


``` java
import java.util.ArrayList;
import java.util.List;

public class 영어끝말잇기 {

	public static void main(String[] args) {
		int n = 5;
		String[] words = {"hello", "observe", "effect", "take", "either", "recognize", "encourage", "ensure", "establish", "hang", "gather", "refer", "reference", "estimate", "executive"};
	
		
		solution(n, words);
	}
	
	static int[] solution(int n, String[] words) {
        int[] answer = new int[2];
        List<String> list = new ArrayList<String>();
        answer[0] = 2; //첫번째 답하는사람은 미리 넣어주기 떄문에 2번째 사람부터 반복문을 돌림
        
        String endWord = words[0].substring(words[0].length()-1,words[0].length());//첫번째 단어의 끝단어 가져오기
        list.add(words[0]);
        int cycle = 1;//처음에는 한번만 순서가 돌았기 떄문에 1초기값 세팅

        for(int i=1; i<words.length; i++) { //2번째 단어부터 loop
        	String frontWord = words[i].substring(0,1); //첫번째 단어 가져옴
        	if(!list.contains(words[i])&& endWord.equals(frontWord)) { //첫번째 단어와 이전 끝단어가 동일하고, 이미 제시한 단어가 아닐 시
        		list.add(words[i]);//제시한 단어는 list에 넣어줌 -> 이미 제시한 단어인지 체크하기 위함
        		if(answer[0]>=n) {
        			answer[0]=1; //한바퀴가 다 돌았을시 1로 세팅하고 cycle+1
        			cycle++;
        		}
        		else {
        			answer[0]++;//아직 한바퀴가 안돌았으면 +1
        		}
        		endWord=words[i].substring(words[i].length()-1,words[i].length());//체크한 단어의 끝 단어 넣어줌
        	}else {
    			answer[1]=cycle; //끝말잇기가 틀렸을 시 answer 2번째 배열에 cycle넣어줌
        		break;
        	}
        }
		if(words.length==list.size()) {//만약 list에 들어간 단어수와 words요청값의 길이가 같으면 한명도 틀리지 않은 것이기 때문에 0으로 세팅
			answer[0] = 0;
			answer[1] = 0;
		}
        return answer;
    }
	

}
```