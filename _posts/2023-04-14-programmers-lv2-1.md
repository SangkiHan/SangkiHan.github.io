---
title: 프로그래머스 무인도여행 LV2
author: SangkiHan
date: 2023-04-14 15:50:00 +0900
categories: [Java, Algorithm]
tags: [Algorithm]
---
------------
섬 전체를 탐색하여 섬들의 합을 찾아야 하기 때문에 DFS를 사용하여 전체를 탐색하여 구현하였다.

``` java
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public class 무인도여행 {
	//방문여부를 담는 check는 전역변수로 지정하였다.
	static boolean[][] check;

	public static void main(String[] args) {
		
		String[] maps = {"X591X","X1X5X","X231X", "1XXX1"};
		solution(maps);
	}
	
	static List<Integer> solution(String[] maps) {
		//return할 답 List
        List<Integer> answer = new ArrayList<>();
		//방문배열 maps와 동일하게 초기화
        check = new boolean[maps.length][maps[0].length()];
        
        for(int i=0; i<maps.length; i++) {
        	for(int j=0; j<maps[0].length(); j++) {
        		//시작지점이 X가 아니거나 방문하지 않았을 시
        		if(!("X".equals(String.valueOf(maps[i].charAt(j)))||check[i][j]==true)){	
        			answer.add(dfs(maps, i, j));
        		}
        	}
        }

		//answer에 담긴 합이 없을시 -1
        if(answer.size()==0) {
        	answer.add(-1);
        }
        
        //내림차순 정렬
        Collections.sort(answer);
        
        return answer;
    }
	
	static int dfs(String[] maps, int x, int y) {
		//x와 y가 maps범위내에서 벗어 났을시 return 0
		if(x==-1|| x==maps.length||y==-1||y==maps[0].length()||check[x][y]==true) {
			return 0;
		}
		else {
			//범위 내지만 만약 X면 return0
			if("X".equals(String.valueOf(maps[x].charAt(y)))) {
				return 0;
			}
			//방문했으니 방문처리
			check[x][y]=true;
		}
		
		//사방으로 dfs돌리기
		return Character.getNumericValue(maps[x].charAt(y))+dfs(maps, x+1, y)+dfs(maps,x,y+1)+dfs(maps,x-1,y)+dfs(maps,x,y-1);
	}
}

```