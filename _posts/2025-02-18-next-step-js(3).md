---
title: 스프레드 활용
author: SangkiHan
date: 2025-02-18 00:20:00 +0900
categories: [NEXT-STEP, JS-TDD-CLEANCODE]
tags: [NEXT-STEP]
---
------------

스프레드 연산자(...)는 배열이나 객체의 요소를 쉽게 펼치고 조작할 수 있는 유용한 기능이다.

## 배열결합

### AS-IS
```javascript
const array1 = [1, 2, 3];
const array2 = [4, 5, 6];

const combinedArray = array1.concat(array2);
console.log(combinedArray); // [1, 2, 3, 4, 5, 6]
```

### TO-BE
``` javascript
const array1 = [1, 2, 3];
const array2 = [4, 5, 6];

const combinedArray = [...array1, ...array2];
console.log(combinedArray); // [1, 2, 3, 4, 5, 6]
```

## 객체병합

### AS-IS
```javascript
const object1 = { a: 1, b: 2 };
const object2 = { b: 3, c: 4 };

const mergedObject = Object.assign({}, object1, object2);
console.log(mergedObject); // { a: 1, b: 3, c: 4 }
```

### TO-BE
``` javascript
const object1 = { a: 1, b: 2 };
const object2 = { b: 3, c: 4 };

const mergedObject = { ...object1, ...object2 };
console.log(mergedObject); // { a: 1, b: 3, c: 4 }
```

## 배열복사

### AS-IS
```javascript
const originalArray = [1, 2, 3];
const copyArray = originalArray.slice(); // 또는 concat 사용
console.log(copyArray); // [1, 2, 3]
```

### TO-BE
``` javascript
const originalArray = [1, 2, 3];
const copyArray = [...originalArray];
console.log(copyArray); // [1, 2, 3]
```
