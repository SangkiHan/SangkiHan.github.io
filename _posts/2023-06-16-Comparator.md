---
title: Comparator를 활용한 리스트 정렬
author: SangkiHan
date: 2023-06-19 10:44:00 +0900
categories: [Java]
tags: [Java]
---
------------

우리는 평소에 int[]나 List<Integer>를 정렬할 시에 java.util.Arrays 와 java.util.Collections을 사용하여 정렬을 한다.  

Ex)
``` java
public class Comparator {
	public static void main(String[] args) {
		Integer[] array = {4, 2, 245, 6, 7, 10, 25, 42123};
		List<Integer> list = new ArrayList<>();
		for(int num : array) {
			list.add(num);
		}
		
		Arrays.sort(array);
		Consumer<Integer>  consumer = s -> System.out.print(s+",");
		List<Integer> list1 = Arrays.asList(array);
		list1.forEach(consumer); 
        //출력 : 2,4,6,7,10,25,245,42123
		
		
		Collections.sort(list);
		System.out.println("\n"+list); 
        //출력 : 2, 4, 6, 7, 10, 25, 245, 42123

        //역순
		Arrays.sort(array, Collections.reverseOrder());
		List<Integer> list2 = Arrays.asList(array);
		list2.forEach(consumer); 
        //출력 : 42123,245,25,10,7,6,4,2
		
		
		Collections.sort(list, Collections.reverseOrder());
		System.out.println("\n"+list); 
        //출력 : 42123, 245, 25, 10, 7, 6, 4, 2
	}
}
```

다만 저런 단순 정렬이 아닌 조건에 의한 정렬일 시는 위와 같은 예로 정렬이 힘들다.  
그럴 때 사용하는 클래스가 Comparator이다.  

먄약 아래와 같은 2차원배열에서 첫번째 정수순으로 정렬해야한다면?  
아래와 같이 해결할 수 있다.
``` java
import java.util.ArrayList;
import java.util.Collections;
import java.util.Comparator;
import java.util.List;

public class ComparatorTest {
	public static void main(String[] args) {
		int[][] array = {`{4, 2},{4, 6},{7, 10},{25, 42123}`};
		
		List<int[]> list = new ArrayList<>();
		list.add(array[0]);
		list.add(array[1]);
		list.add(array[2]);
		list.add(array[3]);
		
		Collections.sort(list, new Comparator<int[]>() {

			@Override
			public int compare(int[] o1, int[] o2) {
				if(o1[0]>o2[0]) {
					return 1;
				}
				else {
					return -1;
				}
			}
		});
		
		for(int[] a : list) {
			System.out.println(a[0]+","+a[1]);
		}
	}
}
```
출력결과

``` text
4,6
4,2
7,10
25,42123
```

근데 여기서 조건이 하나 추가 된다.  
만약 첫번째 정수로 오름차순 정렬을 하되 같을 경우 두번째 정수로 정렬해야한다면?  
아래와 같이 조건을 걸어서 해결 할 수 있다.  
``` java
import java.util.ArrayList;
import java.util.Collections;
import java.util.Comparator;
import java.util.List;

public class ComparatorTest {
	public static void main(String[] args) {
		int[][] array = {`{4, 2},{4, 6},{7, 10},{25, 42123}`};
		
		List<int[]> list = new ArrayList<>();
		list.add(array[0]);
		list.add(array[1]);
		list.add(array[2]);
		list.add(array[3]);
		
		Collections.sort(list, new Comparator<int[]>() {

			@Override
			public int compare(int[] o1, int[] o2) {
				if(o1[0]>o2[0]) {
					return 1;
				}
				else if(o1[0]==o2[0]) {
					if(o1[1]>o2[1]) {
						return 1;
					}
					else {
						return -1;
					}
				}
				else {
					return -1;
				}
			}
		});
		
		for(int[] a : list) {
			System.out.println(a[0]+","+a[1]);
		}
	}
}
```  
출력결과
``` text
4,2
4,6
7,10
25,42123
```

위와 같이 정렬할 시 조건을 걸어서 할 수 있도록 해주는것이 Comperator클래스이다.  
정렬을 할 시 여러 조건이 추가되어야 한다면 Comperator클래스로 해결하도록하자.  

Comperator클래스를 사용한 코딩테스트 문제이다.  
[프로그래머스 카카오 파일명 정렬 Lv2](https://sangkihan.github.io/posts/programmers-lv2-21)
