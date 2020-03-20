---
title: 자바 인터페이스와 추상클래스의 공통점과 차이점
tags: [Java]
style: fill
color: dark
description: 자바의 다형성에서 자주 언급되는 추상클래스와 인터페이스의 공통점과 차이점을 알아보자.
---

## 추상 클래스 VS 인터페이스
명확하게 이해하기 쉬운 것들을 정리하면 아래의 표와 같다. 

| 비교!  | 인터페이스   | 추상 클래스   |
|:-----:|:---------:|:----------:|
| 키워드  |interface  | abstract   |
| 사용성  |implements | extends    |
| 다중성  | 다중 구현 O | 다중 상속 X  |

**추상 클래스**는 하나의 abstract 키워드를 가진 methoad인 **'추상 메소드'가 하나 이상 포함된 클래스**를 말합니다. 추상 메소드가 있으면 클래스는 abstract 키워드를 클래스 앞에 반드시 구현해야 합니다.

```javascript
abstract class A {
	abstract void tank();
}
```

**인터페이스**는 모든 메소드가 추상 메소드로써 구현해야 클래스에서 메소드를 구현의 강제성이 있다. 
```javascript
interface B {
    public void tank1() //Not Error
    public abstract void tank2() // Not Error
    void tank3() // Not Error
    abstract void tank4(); // Not Error
}

public class Test implements B{
    @Override
	public void tank1() { }
    @Override
	public void tank2() { }
    @Override
	public void tank3() { }
    @Override
	public void tank4() { }
}
```
인터페이스의 모든 메소드는 abstract를 명시 안해도 abstract 키워드가 자동적으로 붙는다. 생략되어 있는 상태로 판단하면 된다. class Test implements B 를 하게 되면 tank1, 2, 3, 4를 모두 구현해야하는 것을 확인할 수 있다.

그렇기에 **인터페이스와 추상클래스의 차이를 얘기할 때 추상메소드를 가진 유무로 답을 하면 안된다.** 

## 외형적 말고 의미적 차이점은?
키워드 사용 및 생김새는 다르지만 추상 메소드를 가지는 점에서 비슷한 일을 하긴 한다.

'추상화'란 사물에서 공통된 속성이나 관계 등을 뽑아내는 것이다. 예를 들면 자전거, 차, 비행기, 기차, 버스 등은 모두 탈것(Vehicle)이라는 공통점을 뽑아낼 수 있다. 간단하게 탈 것에는 이름 변수와 이동 메소드를 가지고 있다고 해보자. 탈컷이라는 추상 클래스가 만들어졌고 각 이동매체는 이에 따라 추상 클래스를 상속받고 이동의 방식을 확장하여 사용한다.

즉, 공통화 된 특성/관계를 모아놓은 것이 추상화고 이를 클래스로 만든 것이 추상 클래스라고 생각하면 된다. 아버지 클래스를 만들고 이를 상속받은 아들 클래스를 만드는 것과는 조금은 다르다고 생각하면 된다. **추상화와 상속의 개념을 함께 생각**한다면 개념적 의미를 어느정도 이해할 수 있다.

Collection<E> 와 List<E> 인터페이스를 통해 개념을 알아보자.
```javascript
interface Collection<E> { //다른 코드 생략
    int size()
    boolean isEmpty()
    booelan contains(Objects O)
}
```

Collection 인터페이스에는 이러한 메소드가 존재하고 이것을 구현받아서 만든 인터페이스가 List 인터페이스다. 그리고 이 List 인터페이스를 구현받아서 만든 클래스가 ArrayList, Vector, LinkedList 이다.

List 인터페이스에 구현되어있는 것이 아니라 ArrayList, Vector, LinkedList 클래스에는 size(), isEmpty(), contains(Object O)의 메소드가 구현(인터페이스 구현 강제성)되어 있다. 그리고 구현 받은 객체는 크기, 비어있는지, 어떤 객체를 포함하는지 등의 같은 동작을 수행한다.

즉, **인터페이스 : 그 함수의 구현의 강제성을 통해 같은 동작을 보장하는 것이 목적**

## 결론
- **IT 전공면접에 가면 자주 물어보는 질문이므로 반드시 공통점/차이점을 알고가자**
- 외형적 차이 + 개념적(내부) 차이를 비교해서 이해하기
- 이와 함께 다형성 및 객체지향을 같이 물어보는 경우도 있음