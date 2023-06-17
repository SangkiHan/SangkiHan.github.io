---
title: 프로그래머스 카카오 파일명 정렬 Lv2
author: SangkiHan
date: 2023-06-17 10:44:00 +0900
categories: [Java, Algorithm]
tags: [Algorithm]
---
------------

문자열은 숫자 시작 인덱스와 종료인덱스를 찾아서 잘라줬다.  
정렬은 comparator클래스를 사용하여 조건을 넣어서 정렬을 시도 하여 구현하였다.

``` java
import java.util.ArrayList;
import java.util.Collections;
import java.util.Comparator;
import java.util.List;
import java.util.regex.Pattern;

public class 파일명정렬 {

	public static void main(String[] args) {
		String[] files = {"foo010bar020.zip", "B-50 Superfortress", "A-10 Thunderbolt II", "F-14 Tomcat"};
		solution(files);
	}
	
	
	static String[] solution(String[] files) {
        String[] answer = new String[files.length];
        List<String[]> answerList = new ArrayList<>();
        
        for(String file : files) {
        	String[] sArray = new String[3];
        	int startIndex = 0;
        	int endIndex = 0;
        	for(int i=0; i<file.length(); i++) {
        		char c = file.charAt(i);
        		if(isAlphanumeric(c)) {//숫자 시작 인덱스를 찾는다
        			if(startIndex==0) {
        				startIndex=i;
        			}
        		}
        		else {
        			if(startIndex!=0) {//숫자가 아닌데 이미 시작인덱스가 저장되어있다면 종료인덱스저장
        				endIndex=i;
        				break;
        			}
        		}
        		if(endIndex==0) {
        			endIndex=file.length();
        		}
        	}
        	sArray[0]=file.substring(0,startIndex);//첫번째 문자열은 첫번째 배열에 넣기
        	sArray[1]=file.substring(startIndex,endIndex);//첫번쨰 숫자들은 두변째 배열에 넣기
        	sArray[2]=file.substring(endIndex,file.length());//나머지 뒤에 문자열들은 세번째 배열에 넣기
        	answerList.add(sArray);
        }
        
        Collections.sort(answerList, new Comparator<String[]>() {
			@Override
			public int compare(String[] o1, String[] o2) {
				if(!o1[0].toUpperCase().equals(o2[0].toUpperCase())) {//두 문자열들의 대소문자는 비교하지 않기 때문에 둘다 대문자로 변환 후 비교
					return o1[0].toUpperCase().compareTo(o2[0].toUpperCase());//사전순 정렬
				}
				else {
					if(Integer.valueOf(o1[1])!=Integer.valueOf(o2[1])) {//두 문자열이 같으면 숫자로 비교
						return Integer.valueOf(o1[1])-Integer.valueOf(o2[1]);
					}
				}
				return 0;
			}
		});
        
        for(int i=0; i<answerList.size(); i++) {//answer에 정렬된 문자열들을 붙혀서 넣어줌
        	String[] sArray = answerList.get(i);
        	String s = "";
        	for(int j=0; j<3; j++) {
        		if(sArray[j]!=null||sArray[j]!="") { 
            		s+=sArray[j];
        		}
        	}
        	answer[i]=s;
        }
        
        return answer;
    }
	
	/*
	 * 정규식을 사용하여 숫자인지 판별하는 메소드
	 * */
	static boolean isAlphanumeric(char c) {
	    String regex = "[0-9]";
	    Pattern pattern = Pattern.compile(regex);
	    return pattern.matcher(String.valueOf(c)).matches();
	}
}
```