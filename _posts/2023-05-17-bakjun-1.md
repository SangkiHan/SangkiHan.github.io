---
title: 백준 n과m 
author: SangkiHan
date: 2023-05-17 14:39:00 +0900
categories: [Java, Algorithm]
tags: [Algorithm]
---
------------

### N과M 1
수열이 중복되면 안되지만 원소들은 중복이 되면 되기 때문에 arr배열 전역변수에 값을 계속 바꿔가며 출력해주었다.

``` java
import java.util.ArrayList;
import java.util.List;
import java.util.Scanner;

public class N과M {
	
	static List<List<Integer>> answerList = new ArrayList<>();
	static int[] arr;
	static boolean[] visit;
	
	public static void main(String[] args) {
		Scanner sc = new Scanner(System.in);
		int n = sc.nextInt();
		int m = sc.nextInt();
		
		arr = new int[m];
		visit = new boolean[n];
		
		dfs(n, m, 0);
	}
	
	static void dfs(int N, int M, int depth) {
		if (depth == M) {//최대 깊이까지 탐색햇으면 출력
			for (int val : arr) {
				System.out.print(val + " ");
			}
			System.out.println();
			return;
		}
 
		for (int i = 0; i < N; i++) {
			if (!visit[i]) {//방문안했으면
				visit[i] = true;//방문처리
				arr[depth] = i + 1;
				dfs(N, M, depth + 1);
				visit[i] = false;//방문X
			}
		}
	}
}

```


### N과M 2

1번문제와는 다르게 각 원소들도 중복이 되지않는다, 그렇다는 의미는 1이상 인덱스수가 0인덱스수보다 클수가 없다.  
그렇기 때문에 check라는 방문배열을 만들어 시작 인덱스부터 방문을 하고 출력해주었다.

``` java
import java.util.Scanner;

public class N과M2 {

	public static void main(String[] args) {
		Scanner sc = new Scanner(System.in);
		int n = sc.nextInt();
		int m = sc.nextInt();
		
		dfs(m,0,new boolean[n]);
	}
	
	static void dfs(int m, int index, boolean[] check) {
		if(m==0) {//check에 방문표시가 다 찼으면
			for(int i=0; i<check.length; i++) {
				if(check[i]) {
					System.out.print(i+1+" ");
				}
			}
			System.out.println();
		}
		else {
			for(int i=index; i<check.length; i++) {
				check[i]=true;
				dfs(m-1, index+1, check);
				check[i]=false;
				index++;
			}
		}
	}
}
```


### N과M 3

1번과 다르게 동일한 원소가 배열에 들어갈수있다.  
그래서 방문처리를 따로 하지 않았다.

``` java
import java.util.Scanner;

public class N과M3 {
	
	static int[] arr;
	static StringBuilder str = new StringBuilder();
	public static void main(String[] args) {
		Scanner sc = new Scanner(System.in);
		int n = sc.nextInt();
		int m = sc.nextInt();
		
		arr = new int[m];
		
		dfs(n, m, 0, 0);
		
		System.out.println(str);
	}
	
	static void dfs(int n, int m, int index, int depth) {
		if(m==depth) {//끝까지 탐방하였으면
			for(int i=0; i<m; i++) {
				str.append(arr[i]+" ");
			}
			str.append("\n");
		}
		else {
			for(int i=0; i<n; i++) {
				arr[index]=i+1;
				dfs(n, m, index+1, depth+1);
			}
		}
	}
}
```

### N과M 4

3번과 다르게 1번째 인덱스부터 0번째 인덱스보다 작으면 안되기 때문에 currentNum이라는 변수를 사용하여 작은값이 들어가지 않도록 구현했다.

``` java
import java.util.Scanner;

public class N과M4 {
	static int[] arr;
	static StringBuilder str = new StringBuilder();
	public static void main(String[] args) {
		Scanner sc = new Scanner(System.in);
		int n = sc.nextInt();
		int m = sc.nextInt();
		
		arr = new int[m];//배열수만큼 초기화
		
		dfs(n, m, 0, 0, 1);
		
		System.out.println(str);
	}
	
	static void dfs(int n, int m, int index, int depth, int currentNum) {
		if(m==depth) {//끝까지 탐색했을시
			for(int i=0; i<m; i++) {
				str.append(arr[i]+" ");
			}
			str.append("\n");
		}
		else {
			for(int i=0; i<n; i++) {
				if(currentNum>i+1) {
					continue;
				}
				arr[index]=i+1;
				dfs(n, m, index+1, depth+1, i+1);
			}
		}
	}
}
```