---
title: Jest 활용
author: SangkiHan
date: 2025-02-11 11:20:00 +0900
categories: [NEXT-STEP, JS-TDD-CLEANCODE]
tags: [NEXT-STEP]
---
------------

## Jest
Jest는 JavaScript 및 TypeScript로 작성된 애플리케이션을 위한 오픈 소스 테스트 프레임워크입니다. Facebook에서 개발하였으며, 주로 유닛 테스트에 사용됩니다.

## 사용사례
아래는 간단한 자동차게임에 필요한 객체이다.

### Car.js
```javascript
import * as rand from "../util/random.js"

class Car {

    static MINIMUM_LENGTH = 1;
    static MAXIMUM_LENGTH = 5;
    static OVER_MINIMUM_LENGTH_MESSAGE = "자동차 이름은 1자 이상이여야합니다."
    static OVER_MAXIMUM_LENGTH_MESSAGE = "자동차 이름은 5자를 넘으면 안됩니다."

    constructor(name) {
        this.confirmMinimumLength(name);
        this.confirmMaximumLength(name)
        this.name = name;
        this.distance = 0;
    }

    moveForward() {
        if(rand.randMoveForward(10, 4)) {
            this.distance += 1;
        }
    }

    confirmMinimumLength(name) {
        if(name.length < Car.MINIMUM_LENGTH) {
            throw new Error(Car.OVER_MINIMUM_LENGTH_MESSAGE)
        }
    }

    confirmMaximumLength(name) {
        if(name.length > Car.MAXIMUM_LENGTH) {
            throw new Error(Car.OVER_MAXIMUM_LENGTH_MESSAGE)
        }
    }
}

export default Car;
```

beforeEach를 이용하여 초기 데이터를 공통적으로 생성  
expect를 이용하여 원하는 값을 검증  
toThrow를 이용하여 예측 가능한 예외를 검증하였다.

### Car.test.js
``` javascript
import Car from "../src/domain/car.js";

describe("Car Class 테스트", () => {
    let car;

    beforeEach(() => {
        jest.clearAllMocks(); 
        jest.spyOn(Math, "random").mockReturnValue(0.4);
        car = new Car("NEXT");
    });

    it("자동차는 이름을 가진다.", () => {
        const car = new Car("NEXT");
    
        expect(car.name).toEqual("NEXT");
    
    })
    
    it("자동차는 초기에는 거리 0을 가진다.", () => {
        const car = new Car("NEXT");
    
        expect(car.distance).toEqual(0);
    })
    
    it("자동차를 전진시킨다.", () => { 
        const car = new Car("NEXT");
    
        car.moveForward();
    
        expect(car.distance).toEqual(1);
    })
    
    it("자동차는 5자리를 넘으면 안된다.", () => {
        expect(() => new Car("NEXTSTEP")).toThrow(Car.OVER_MAXIMUM_LENGTH_MESSAGE);
    })
    
    it("자동차는 1자리를 이상이여야 한다.", () => {
        expect(() => new Car("")).toThrow(Car.OVER_MINIMUM_LENGTH_MESSAGE);
    })
})
```