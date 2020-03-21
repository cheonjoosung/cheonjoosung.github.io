---
title: 자바 상속의 개념 그리고 예제/활용
tags: [Java]
style: fill
color: dark
description: 자바에서 필수 개념 중 하나인 상속과 이와 관련된 super, this 등의 활용법을 알아보자
---

## 상속이란 ?
상속은 영어로 Inheritance 이고 사전적 의미는 일정한 친족 관계가 있을 때 A가 가진 권리/의무의 일체를 다른 사람에게 전달해주는 일을 말한다.

이 개념을 코딩으로 가져오면 권리/의무 함수가 가지고 있는 변수 또는 메소드라고 생각하면 된다. 

예를 들면
- 자전거는 타는 것중의 한 종류다
- 경찰차는 차다.
- 전동 퀵보드는 이륜 차다.

상속의 개념을 Class 로 표현해보자
```javascript
public class 차 { }

public class 경찰차 extends 차 { }
```

자바에서는 상속을 활용하기 위해서 extends 키워드를 사용하여 부모 클래스를 적고 가진 정보들을 상속 받는다. 즉, 자식은 부모가 가진 것을 사용할 수 있게 된다.

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
        super.run();
    }
}

경찰차 _경찰차 = new 차()
_경찰차.siren() //삐용 삐용
_경찰차.run()   //달려
```

경찰차 클래스에는 siren() 메소드 밖에 없지만 부모인 차를 상속받았기에 부모의 메소드인 run() 메소드를 사용할 수 있다. run() 메소드를 override 한 후에 확장을 한 후에 자유롭게 쓸 수 있다. 위의 super.run()은 아래에서 설명할께요.

## 접근 제한자
접근 제한자는 public, private, protected, default 가 있고 **상속과 관련된 것은 protected** 이다. 
- public    : 어떤 클래스에서도 접근 가능
- protected : 패키지가 달라도 **상속받은 자식 클래스에서는 접근 가능**
- private   : 자기 클래스만 가능 -> 내부 변수는 private으로 선언하고 get/set을 통해 접근 및 설정 -> **정보 은닉**
- default   : 같은 패키지에서만 접근

## 추상 클래스
추상 클래스는 abstract method 하나 이상 가진 클래스를 의미합니다. 외형적 의미보다는 개념적 의미가 중요하기에 [추상 클래스 vs 인터페이스](java-interface-abstract) 포스트를 통해 읽어보시거나 검색을 통해 '추상화' 또는 '다형성'에 대한 학습이 필요합니다.

## super, super(), this, this() ?
super 는 부모 클래스를 지칭하고 super() 부모 클래스의 생성자를 호출하기 위함이다. 반면에 this 는 자기 자신 클래스를 의미하고 this() 자기 자신 클래스의 생성자를 호출할 때 사용한다.

예제를 살펴보자
```javascript
class A {
	private String name;
	private int age = 99;
	
	A(int age, String name) {
		this.age = age;
		this.name = name;
	}
	
	A(int age) {
		this(age, "Joo");
	}
	
	public void run() { 
		System.out.println("달려"); 
	}
}

class B extends A {
	B(int age, String name) {
		super(age, name);
	}
	B(int age) {
		super(age);
	}
	
	public void siren() { 
		System.out.println("삐");
	}
	@Override
	public void run() {
		super.run();
	}
}
```

클래스 A에는 age,name을 받는 생성자와 age 만 받는 생성자 2개가 있다. 둘 다의 정보가 필요하기에 age만 입력하여 호출했을 때 this(age, "joo")를 통해 자기 자신의 다른 생성자를 호출할 수 있고 this. 를 활용하여 자기 자신의 변수에 접근이 가능하다.

클래스 B에의 생성자에서 super(age, name)을 통해 부모의 생성자를 호출할 수 있다. 또한, 오버라이딩한 메소드인 run() 에서 super.run()에서 super. 부모의 클래스를 호출하고 그중 run()에 접근할 수 있다.


## 클래스 형변환
부모의 타입으로 자식의 객체를 참조하게 되면 부모의 메소드만 사용이 가능하다. 그렇기에 자식 객체의 속성이나 메소드를 사용하기 위해서는 형변환이 필요하다.

예제를 살펴보자
```javascript
class A {
	public void run() { 
		System.out.println("달려"); 
	}
}

class B extends A {
	public void siren() { 
		System.out.println("삐");
	}
}

A a = new B();
a.run() //정상 작동
a.siren() //컴파일 에러 발생
((B) a).siren() //정상 작동

B b = (B) a
b.run()
b.siren()
```

타입 캐스트를 통해 변환 후 siren()에 접근을 해도 되고 B b 의 객체를 만든 뒤에 a 를 할당해도 된다. 그 반대의 경우 **에러가 발생하므로 주의**해야 함.

여기에서 의문이 생길 수 있겠죠? Why ? 어째서 ? A a = new B()로 선언을 해야 하는가? B b = new B()로 선언하면 편한 것 같은데... 이러한 의문은 추후에 **다형성에 관한 포스트**를 올리겠습니다.


## 결론
- **IT 전공면접에 가면 자주 물어보는 질문이므로 반드시 공통점/차이점을 알고가자**
- 외형적 차이 + 개념적(내부) 차이를 비교해서 이해하기
- 이와 함께 다형성 및 객체지향을 같이 물어보는 경우도 있음