---
title: 프로그래머스 괄호회전하기 Lv2
author: SangkiHan
date: 2023-09-24 15:15:00 +0900
categories: [Java, Algorithm]
tags: [Algorithm]
---
------------

Stack의 LIFO(Last In First Out) 특성을 이용하여 오른쪽으로 열린 괄호일시 Stack에 반대괄호를 넣어두고 하나씩 꺼내어 검증하는 식으로 구현하였다.  

``` java
public class 괄호회전하기 {
	
	static int answer = 0;
	static Stack<Character> stack = new Stack<>();
	static HashMap<Character,Character> hash = new HashMap<>();

	public static void main(String[] args) {
		
		String s = "}]()[{";
		hash.put('[', ']'); //괄호 초기값 세팅
		hash.put('(', ')');
		hash.put('{', '}');
		
		solution(s);
		System.out.println(answer);
	}
	
	static void solution(String s) {
		for(int i=0; i<s.length(); i++) {
			if(check(s)) {
				answer++;
			}
			s = s.substring(1,s.length()) + s.substring(0,1); //첫번째 문자 맨 뒤로 이동
		}
	}
	
	static boolean check(String s) {
		boolean check = true;
		for(int i=0; i<s.length(); i++) {
			char c = s.charAt(i);
			if(c=='[' || c=='{' || c=='(') { //오른쪽으로 열린 괄호일시 Stack에 반대 괄호 넣어둠
				stack.add(hash.get(c));
			}
			else {
				try { //Stack이 비어있을 경우 예외 처리 후 false
					char sc = stack.pop(); //Stack에서 마지막으로 들어간 값 가져옴
					if(sc!=c) {
						check = false;
						break;
					}
				}
				catch (EmptyStackException e) {
					check = false;
					break;
				}
			}
		}
		
		if(stack.size()>0) { //Stack에 괄호가 남아있으면 완전하지 않는 괄호이기에 false
			check = false;
		}
		
		return check;
	}

}

```