---
title: map 메소드 활용
author: SangkiHan
date: 2025-02-17 02:20:00 +0900
categories: [NEXT-STEP, JS-TDD-CLEANCODE]
tags: [NEXT-STEP]
---
------------

### AS-IS
```javascript
const cars = [];
for (const name of names) {
    cars.push(new Car(name));
}
```

### TO-BE

map을 이용하여 간단한 반복문은 더욱 간결하고 가독성있게 표현할 수 있다.

``` javascript
const cars = names.map(name => new Car(name))
```