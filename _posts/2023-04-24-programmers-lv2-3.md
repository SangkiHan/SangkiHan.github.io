---
title: 프로그래머스 마법의 엘레베이터 Lv2
author: SangkiHan
date: 2023-04-24 16:50:00 +0900
categories: [Java, Algorithm]
tags: [Algorithm]
---
------------

1번째 자리수가 5이상일시 + 5이하일시 -하는식으로 풀이했다.  
그랬더니 테스트케이스 4개가 통과하지 못했다.  
그래서 반례들을 생각해보니  
1번째 자리수가 5이고 2번째 자리수가 5이상일시 +하여 2번째 자리수를 더해주는것이 횟수를 줄일 수 있는 방법이라는 것을 찾아냈다.

``` java
public class 마법의엘레베이터 {

	public static void main(String[] args) {
		
		int strey = 555;
		
		solution(strey);
	}
	
	// 마법의 돌 종류 : 10의 n승
	static int solution(int storey) {
        int answer = 0;
        
        while(storey!=0) {//요청값이 0이 될 때 까지 반복문
        	int a = storey%10;//1자리수 가져오기
        	if(a>5) {//5이상이면 +하는게 이득
        		answer+=(10-a);
        		storey+=10;
        	}
        	else if(a<5){//5이하면 -하는게 이득
        		answer+=a;
        	}
        	else{						
        		if((storey/10)%10>4) {//1자리수가 5이고 그 다음 자리수가 5이상일시는 +를 하여 다음자리수를 한자리 올려주는게 이득
            		storey+=10;
        		}
    			answer+=(10-a);//요청값 storey가 한자리수 일때는 (storey/10)%10>4 이 조건을 충족시키기 않기 때문에 밖에다가 +해줬다.
        	}
        	storey/=10;//storey에 다음자리수부터 초기화
        }
        return answer;
    }
}
```