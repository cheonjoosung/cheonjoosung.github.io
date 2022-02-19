---
title: (Java) String, StringBuffer, StringBuilder 공통점 및 차이점
tags: [Java]
style: fill
color: dark
description: 안드로이드 버전을 출시할 때 로그를 하나씩 지우는게 아니라 디버그 모드에서만 작동되도록 만들기
---

## String
String Class는 문자열 관련 기능을 작업할 때 유용한 메소드를 제공한다.

```javascript
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence,
               Constable, ConstantDesc

    @Stable
    private final byte[] value;
```
스트링 클래스는 final로 선언되어 있어서 상속받아 새로운 기능을 만들 수 없다. 또한, 인스턴스가 한 번 생성되면 읽기만 가능하고 변경이 불가능하다. 리터럴(””)이나 new 키워드를 사용하여 문자열을 할당하면 내부적으로 final 로 선언된 변수에 저장이 된다.


**Equals & ==**
```javascript
String str = "str";
String str2 = "str";
String str3 = new String("str");

System.out.println(str == str2); // true
System.out.println(str == str3); // false
System.out.println(str2 == str3); // false

System.out.println(str.equals(str2)); // true
System.out.println(str.equals(str3)); // true
System.out.println(str2.equals(str3)); // true
```
리터럴(””)를 활용하여 문자열을 할당하거나 new 키워드를 활용하여 문자열을 할당한다. 자바에서는 문자열을 비교할 때 equals을 사용해야 한다.

== 은 객체의 주소값을 비교하므로 str과 str2의 비교는 false가 나와야 하는데 true가 나온다. 다른 객체인데도 동일한 주소값을 가진다. str3은 str1과 str2와는 같은 주소값을 가지고 있지 않다.

**→ new 키워드를 사용했을 때와 = 사용했을 때 내부적으로 동작하는 로직이 다르다는 것이다.**


**문자열 메모리 영역 할당**
자바에서 변수를 할당하면 Java Heap 영역에 할당이 된다. 문자열은 조금 다른 부분이 있다.
```javascript
String str1 = "madplay"
String str2 = "madplay"
String str3 = new String("madplay")
String str4 = new String("madplay")
```
![preview](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/15dfa707-c545-432a-a8f2-121df68af8fc/Untitled.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45EIPT3X45%2F20220219%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20220219T122816Z&X-Amz-Expires=86400&X-Amz-Signature=394d21b59c7ac41ba6001db3bd254473747733bbe8f65e0a724550f71103a753&X-Amz-SignedHeaders=host&response-content-disposition=filename%20%3D%22Untitled.png%22&x-id=GetObject)
str1을 할당한 후 str2의 동일한 값을 할당한 경우 동일한 주소값을 참조하고 있다. String constant pool에 의해서 이미 생성되어있는지 확인한 후에 있으면 그 값을 참조하도록 하고 있다. String Class의 intern() 메소드가 풀을 검색하고 매핑을 하는 역할을 한다.

new 키워드를 생성했을 경우 constant pool이 아닌 영역에 매번 생성하고 있다.

**→** 리터럴(””) **과 new 키워드를 사용했을 때 메모리 할당되는 방식이 다르다.**

→ 문자열을 사용한다면 리터럴(””)을 사용하는 것이 그나마 메모리 낭비를 줄일 수 있다.


**Immutable 불변객체라고 했는데 우리는 문자열 변수를 변경을 자주 했었다?**
```javascript
String a = "Hello"
a += " World"
```
final 상수라고 선언했지만 우리는 아주 자연스럽게 값을 변경했지만 컴파일 & 런타임시 아무런 문제가 발생하지 않는다.
![preview](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/2e8f23b0-a4a4-43fb-806a-933a05e084cf/Untitled.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45EIPT3X45%2F20220219%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20220219T122820Z&X-Amz-Expires=86400&X-Amz-Signature=aafd1ab6706dec2d88816aba979a2ae30b2897cb8ee810e11bb8dfbef68e44a9&X-Amz-SignedHeaders=host&response-content-disposition=filename%20%3D%22Untitled.png%22&x-id=GetObject)
a += “ World” 를 했을때 기존값에 변경이 일어날거라고 생각을 하지만 final로 선언되었기에 변경자체가 불가능하다. Hello World라는 값을 하나 만든후에 참조하는 값을 변경한다.

**String constant pool에 있는 “Hello”는 Heap 영역에 있고 사용하지 않는 다면 언젠가 garbage Collect가 처리해줄 것 이다.**


**String 적절한 사용이 필요하다.**
리터럴(””)을 사용하여 문자열을 만들면 String constant pool에 만들어진다. 누군가 해당 문자열을 사용할 때 그대로 사용이 가능하기에 메모리의 낭비를 어느 정도 줄일 수 있다

❓ 그렇다면 매번 “”을 사용하여 문자열을 쓰는 것이 옳을까에 대한 고민을 해봐야 한다.
```javascript
String APPLICATION_NAME = "TEST"

String a = ""
for (int i=0 ; i<1000 ; i++)
	a += "++더하기++"
```
위의 코드에서 Application Name은 변경이 자주 일어나지 않고 다른곳에서 자주사용하기에 문자열로 사용해도 큰 문제는 없을 것이다.

하지만, a += “++더하기++” 의 경우 String Constant Pool에 게속해서 쌓일 것이고 자주사용하지 않는 변수로 메모리 공간을 낭비한다. 예제 코드에서는 1000개 까지만 했지만 더 많이 쓰인다면 성능 이슈가 발생할 것이다.

**→ 방법은 문자열 변경/수정이 자주 발생한다면 StringBuilder/StringBuffer 를 사용해야 한다.**


## StringBuilder & StringBuffer
StringBuilder & StringBuffer 는 문자열의 추가/변경할 때 쓰는 자료형이다.

**사용 및 메소드**
```javascript
StringBuilder stringBuilder = new StringBuilder("String Builder");
stringBuilder.append("1").append(1).append('1');
stringBuilder.deleteCharAt(1);
stringBuilder.indexOf("1");
stringBuilder.deleteCharAt(1);
stringBuilder.toString();

StringBuffer stringBuffer = new StringBuffer("String Buffer");
stringBuffer.append("1").append(1).append('1');
stringBuffer.deleteCharAt(1);
stringBuffer.indexOf("1");
stringBuffer.deleteCharAt(1);
stringBuffer.toString();
```
두 자료형 AbstractStringBuilder를 상속받아 구현하기에 메소드를 사용하는 방법이 거의 유사하다.

생성자를 활용하여 초기값을 할당해주는데 내부적으로 저장하는 형태는 가변 배열이다.

대표적인 추가 append, 삭제 delete, 삽입 insert, 찾기 indexOf, 변환 toString 이외에도 다양한 메소드가 있다.

```javascript
@Override
public StringBuilder append(char[] str, int offset, int len) {
    super.append(str, offset, len);
    return this;
}

@Override
public synchronized StringBuffer append(CharSequence s) {
    toStringCache = null;
    super.append(s);
    return this;
}
```
append 뿐만 아니라 다른 메소드도 자신 객체를 리턴하는 Builder Pattern을 사용한다. sb.append().append().append()를 이어붙어서 문자열 관리에 쉽게 사용이 가능하다.

String 자료형과 달리 변수를 추가할 때 힙 영역에 계속해서 추가하는 것이 아니라 해당 주소가 참조하는 값을 바꾸기 때문에 메모리 낭비가 적다.

**두개의 차이점 - “Thread Safe”**
StringBuilder 와 StringBuffer는 상속받는 추상클래스도 같고 비슷한 메소드를 사용한다. 그러면 어느 상황에 시기적절하게 사용해야 할까?
```javascript
@Override
public StringBuilder append(char[] str, int offset, int len) {
  super.append(str, offset, len);
  return this;
}

@Override
public synchronized StringBuffer append(CharSequence s) {
  toStringCache = null;
  super.append(s);
  return this;
}
```
StringBuilder와 StringBuffer의 각 메소드를 보면 분명한 차이점이 하나 있.다 StringBuffer 에는 **“synchronized”라는 키워드**가 append뿐만 아니라 다른 메소드도 쓰고 있다.

StringBuffer는 synchronized 키워드를 통해 멀티쓰레드 환경에서 안전하게 사용할 수 있다. **하지만, 해당영역의 락을 걸고 푸는 과정에서 오버헤드가 발생**하기에 필요한 경우를 잘 구분해야 한다.

‼️ **StringBuffer 사용**

- 멀티 쓰레드 환경에서 안전한 프로그램
- static 선언된 문자열 변경
- singleton으로 선언된 클래스의 문자열 변경

‼️ **StringBuilder 사용**

- 단일 스레도 사용으로 쓰레드 안전 여부 불필요 시
- 메서드 내에서 지역 변수로 사용 시

2개의 쓰레드가 있고 쓰레드는 문자열에 c가 있으면 뒤에 c를 그 숫자만큼 붙인다고 가정해보자.

String a = “abc” 두 번을 반복했을 때 결과는 abc → abcc → abcccc가 되어야 한다.

1번 쓰레드는 c가 있으면 있는 수만큼 d를 가 문자열을 수정할때 abcc가 되어야 한다.

2번 쓰레드가 실행할 때 1번 쓰레드의 결과인 **“abcc”**를 받아서 할 수도 있지만, 1번 쓰레드의 해당 작업이 끝나기 전에 2번 쓰레드가 참여하면 **“abc”**를 바탕으로 작업이 진행될것이고 최종 변경된 값은 **“abcccc”**가 아니라 **“abcc”**가 나온다.

→ **쓰레드가 언제 실행될지 예측이 정확히 안되기에 메소드 락(상호 베타적 인 엑세스)을 방지하여 동시 프로그래밍의 오류를 방지해야 한다.**


## 결론
- 때와 시기적절하게 문자열 클래스를 사용하자
