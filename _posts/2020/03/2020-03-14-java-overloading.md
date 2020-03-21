---
title: 자바 오버라이딩과 오버로딩 개념과 예제
tags: [Java]
style: fill
color: dark
description: 오버라이딩과 오버로딩의 개념과 차이를 이해하고 자바의 다형성을 파악해보기
---

## 오버로딩(Overloading)
함수로 전달되는 매개변수의 유형과 개수가 다르게 하여 동일한 이름의 여러 메소드를 가지게 하는 기법

**메소드 오버로딩**을 덧셈 기능을 통해 알아보자
```javascript
sum(1 + 4)
sum(1.0 + 4.0)

int sum(int a, int b) {
	return a+b
}

double sum(double a, double b) {
	return a+b
}
```

계산기의 덧셈 기능을 만든다고 가정을 하면 인자가 정수형이 올 수도 있고 실수형이 올 수도 있다. 이럴 때 사용할 수 있는 기법이 메소드 오버로딩이다. 메소드 sum()에서 인자를 (int,int) 또는 (double,double) 에 따라 자바 컴파일러가 인식을 해서 매칭시킨다.


**생성자 오버로딩**
생성자도 매개변수의 유형과 개수를 달리하여 동일한 이름을 가지는 생성자를 여러개 만들 수 있다.

```javascript
class A {
	private String name;
	private int age = 99;
	
	A() {
		this(0, "no name")
	}

	A(int age, String name) {
		this.age = age;
		this.name = name;
	}
	
	A(int age) {
		this(age, "no name");
	}
}

A a = new A();
A a = new A(50);
A a = new A(50, "WHAT")
```

a 객체를 만들 때 인자의 유형과 갯수에 따라 다양하게 생성자를 호출하여 사용할 수 있다. 예를 들면 객체에 정보를 넣을 때 특정 속성이 없는 경우가 있기에 생성자를 여러개 만들어 적절하게 호출할 수 있다. 위의 코드에서는 age나 name의 인자가 없는 경우 [this() 키워드](java-inherit)를 통해 자기 자신의 다른 생성자를 호출했다.


## 오버라이딩(Overriding)
오버라이딩은 부모의 메소드와 똑같은 타입&이름을 가진 메소드를 자식이 가지는 것으로 기능을 확장해서 사용할 수 있다. 오버로딩(Overloading)이 메소드의 편리한 활용이라면 오버라이딩(Overriding)은 [상속의 개념](java-inherit)과 관련이 있다.

아래의 코드를 통해 오버라이딩을 살펴보자.
```javascript
public class 차 {
    public void run() {
        print("달려")
    }
 }

public class 경찰차 extends 차 {
    public void siren() {
        print("삐용 삐용")
    }

    @Override
    public void run() { //확장을 원한다면 ?
		//super.run()
        print("경찰차가 달린다")
    }
}

경찰차 _경찰차 = new 차()
_경찰차.run()   //
```
일단 오버라이딩을 하게 되면 메소드 위에 @Override 라고 표시가 만들어 진다.

부모 클래스인 차에서 run()dms "달려"만 출력한다. 하지만 자식 클래스인 경찰차에서는 run() 메소드를 오버라이딩 한 후에 "경찰차가 달린다"을 출력하는 것으로 기능을 확장시켰다. 


## 예제
```javascript
class A {	
	public void run() { 
		System.out.println("A : RUN()"); 
	}
}

class B extends A {
	public void siren() { 
		System.out.println("B : SIREN()");
	}
	
	@Override
	public void run() {
		System.out.println("B : RUN()");
	}
}

//1번
A a1 = new A();
a1.run();

//2번	
B b1 = new B();
b1.siren();
b1.run();

//3번	
A a2 = new B();
a2.run();
a2.siren(); //Compile Error
((B)a2).siren();

//4번	
B b2 = new A();    //Compile Error
B b2 = (B) new A();//Runtime Error
b2.run();
```

4가지의 경우를 살펴보면 1, 2번의 경우는 눈에 보이는대로 따라가면 찾을 수 있다. 

3번의 경우는 자식을 참조하게 했지만 컴파일러에서는 보이지 않으므로 타입 캐스트를 통해 알려줘야 합니다. ((B)a2).siren() 을 하면 정상적으로 메소드를 사용할 수 있습니다.

4번의 경우는 컴파일 에러가 나고 타입 캐스트를 해도 안됩니다. 부모 객체 = new 자식의 경우만 정상적인 작동이 가능합니다.

```javascript
class A {
	public A() {
		System.out.println("부모 호출");
	}

	public void run() { 
		System.out.println("A : RUN()"); 
	}
}

class B extends A {
	public B() {
		System.out.println("자식 호출");
	}

	public void siren() { 
		System.out.println("B : SIREN()");
	}
}

//1번
B b1 = new B();
b1.siren();
b1.run();

//2번	
A a2 = new B();
a2.run();
((B)a2).siren();
```

1번의 경우
- 부모 호출 -> 자식 호출 -> B SIREN() -> A RUN() 
- new B()를 할 때 부모가 먼저 불린다. B의 생성자 안에는 super()가 생략되어 있다고 생각하면 된다.

2번의 경우
- 부모 호출 -> 자식 호출 -> A RUN() -> B SIRENN()
- 1번의 경우가 동일하다.

## 결론
- **IT 전공면접뿐만 아니라 필기시험에도 빈출되는 문제**
- 오버라이딩이 길다 -> 대대손손 남겨야 한다 -> 상속.. 가끔씩 햇갈린다면 이렇게 연상하는 것도 방법입니다.