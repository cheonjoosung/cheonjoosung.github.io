---
title: Java Reference (Soft vs Strong)
tags: [Java, Android]
style: fill
color: dark
description: (Java) Java Reference (Soft vs Strong)
---

## Reference
Reference(참조)란 무엇일까?

사전적 의미로는 어떤 것에 대해 말하기, 정보를 얻기 위한 참고/참조, 문의 조회가 있다. Java 에서는 정보를 얻기 위한 참조가 가장 가까운 의미라고 생각한다.

아래 코드를 보자

```javascript
Parent a;

a.print() // a is not initiallized
a = new Child()
```
첫 줄만 쓰고 a 변수를 사용할 수 없다. a의 메소드에 접근하려고 하면 초기화가 되어있지 않다고 나온다. 객체를 사용하기 위해서 new 키워드를 사용하고 내부적으로 heap 영역에 할당이 된다. a라는 **정보를 접근하기 위해서 해당 메모리를 참조/참고**한다고 생각하면 된다.

안드로이드를 개발하면 짧은 시간 가끔 쓰이는 객체를 사용할 때가 있다. 예를 들면 앱을 시작했을 때 변조/해킹 방지, 이미지 로딩 프로세스 등이 있다고 해보자. 이들은 호출이 되는 시점에 짧은시간 동작을 하고 끝이 난다. Heap 영역에는 사용하지 않는 공간을 빈도가 낮고 짧은 시간을 사용하는 객체가 차지하고 있다라는 점이다.

→ 메모리 누수가 많이 되어 OOM(Out of Memory) 현상이 발생되어 애플리케이션이 종료되는 현상이 발생할 수 있다.

**StrongReference**
```javascript
Parent a = new Child()
```
- new 키워드를 통해 생성된 객체
- GC의 제외대상이기에 메모리 누수를 방지하기 위해서 안쓰는 경우 참조를 지워줘야 함

**SoftReference**
```javascript
SoftReference<T> softRef = new SoftReference(new T());
```
- SoftRefernce 키워드를 사용하여 사용하고자 하는 타입을 전달
- 메모리가 충분하지 않을 때 GC에 의해 수거

**WeakReference**
```javascript
WeakReference<T> weakRef = new WeakReference(new T());
```
- SoftRefernce 와 유사하지만 GC가 발생하면 무조건 회수
- 위에서 언급한 짧은 시간 자주 쓰이는 객체를 사용할 때 유용

**PhantomReference**
```javascript
PhantomReference(T referent, ReferenceQueue<? super T> q)
```
- 예제도 거의 없을 정도로 사용 빈도가 적다

## 결론
- 자주 짧게 쓰이는 객체가 있을 때 WeakReference를 사용하자
