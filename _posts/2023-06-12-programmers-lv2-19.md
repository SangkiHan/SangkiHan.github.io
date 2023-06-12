---
title: 프로그래머스 주차요금계산 Lv2
author: SangkiHan
date: 2023-06-12 17:44:00 +0900
categories: [Java, Algorithm]
tags: [Algorithm]
---
------------

차량입차정보를 어떻게 저장을 해야할까 고민하다가, CarInfo라는 객체를 생성해서 풀이 하였다. HashMap에 Key값으로 차량번호 value로 차량입차정보를 담아 풀이하였다.

``` java
import java.util.ArrayList;
import java.util.HashMap;
import java.util.Iterator;
import java.util.List;
import java.util.Map;
import java.util.TreeMap;

public class 주차요금계산 {
	
	static int basicCharge = 0;
	static int basicTime = 0;
	static int partTime = 0;
	static int partCharge = 0;
	
	public static void main(String[] args) {
		int[] fees = {180, 5000, 10, 600};
		String[] records = {"05:34 5961 IN", "06:00 0000 IN", "06:34 0000 OUT", "07:59 5961 OUT", "07:59 0148 IN", "18:59 0000 IN", "19:09 0148 OUT", "22:59 5961 IN", "23:00 5961 OUT"};
		solution(fees,records);
	}
	
	static List<Integer> solution(int[] fees, String[] records) {
        List<Integer> answer = new ArrayList<Integer>();//정답리스트
        HashMap<String, CarInfo> carMap = new HashMap<String, CarInfo>();//출차 관리 Map
        Map<String, Integer> carTime = new TreeMap<>();//각 차량 누적시간 key로 자동정렬 할 수 있도록 treeMap사용
        
        basicTime = fees[0];//기본시간
        basicCharge = fees[1];//기본요금
        partTime = fees[2];//단위시간 
        partCharge = fees[3];//단위요금
        
        for(String record : records) { 
        	String[] car = record.split(" ");//시간, 차량번호, 출차여부 쪼개기
        	String recordTime = car[0];//시간
        	String[] hourMinute = recordTime.split(":");//시 분으로 다시 쪼개기
        	String carNum = car[1];//차량번호
        	String inOut = car[2];//출차여부
        	CarInfo carInfo = new CarInfo(Integer.parseInt(hourMinute[0]) , Integer.parseInt(hourMinute[1]));//차량 생성자로 객체생성
        	if("IN".equals(inOut)) {//들어왔을시 carMap에 넣어줌
        		carMap.put(carNum, carInfo);
        	}
        	else {//출차시 carMap에 들어있던 입차시간과 대조
        		CarInfo inCarInfo = carMap.get(carNum);
        		int time = calTime(carInfo, inCarInfo);
        		carMap.remove(carNum);//시간계산 완료후 없애줌
        		carTime.put(carNum,carTime.getOrDefault(carNum,0)+time);//시간넣어줌
        	}
        }
        
        Iterator<String> iterator = carMap.keySet().iterator();
        while(iterator.hasNext()) {//carMap에 출차기록이 없는 차량은 23시59분으로 차량임의생성하여 계산
        	String carNum = (String)iterator.next();
        	CarInfo inCarInfo = carMap.get(carNum);
        	CarInfo carInfo = new CarInfo(23 , 59);
    		int time = calTime(carInfo, inCarInfo);
    		carTime.put(carNum,carTime.getOrDefault(carNum,0)+time);
        }
        
        Iterator<String> iterator2 = carTime.keySet().iterator();
        while(iterator2.hasNext()) {//차량시간들 뽑아서 요금계산하여 answer에 넣어줌
        	String carNum = (String)iterator2.next();
        	int time = carTime.get(carNum);
    		answer.add(charge(time));
        }
        
        return answer;
    }
	/*
	 * 시간계산 메소드
	 * */
	static int calTime(CarInfo outCarInfo, CarInfo inCarInfo) {
		return (((outCarInfo.getHour()-inCarInfo.getHour())*60)+(outCarInfo.getMinute()-inCarInfo.getMinute()));
	}
	/*
	 * 요금계산 메소드
	 * */
	static int charge(int time) {
		return (int) ((time>basicTime) ? basicCharge+((int) Math.ceil(((double)(time-basicTime)/(double)partTime)) *partCharge) : basicCharge);
	}
	
	/*차량객체*/
	static class CarInfo {
		private int hour;
		private int minute;
		
		public CarInfo(int hour, int minute) {
			super();
			this.hour = hour;
			this.minute = minute;
		}
		
		public int getHour() {
			return hour;
		}
		public int getMinute() {
			return minute;
		}
	}
}
```