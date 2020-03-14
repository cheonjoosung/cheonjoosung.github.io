---
layout: post
title:  "Java Static Factory Method"
date:   2018-12-08
categories: java
highlight: false
image: /images/it/java.png
tag: java
---

###### 클래스를 이용하여 객체를 만들 때 일반적으로 public으로 선언된 생성자를 이용합니다. 이외에도 public으로 선언된 정적 팩터리 메서드(static factory method) 를 추가하는 방법이 있습니다. Design Pattern 의 팩터리 메서드의 개념과 다름의 유의!

<br>
# 장점
---------------------
###### 생성자 대신 정적 팩터리 메서드를 사용하면 다양한 장점이 생깁니다.

**1. 정적 팩터리 메서드에 이름이 존재**
* 이름으로 접근하기에 사용성을 쉽게 파악할 수 있고 가독성 또한 증가
* 같은 시그니처를 갖는 생성자를 여러 개 정의할 필요가 있을 때 팩토리 메서드 사용


**2. 새로운 객체를 생성할 필요가 없음**
* 만든 객체를 캐시 해놓고 재사용하여 불필요한 생성 방지 -> 성능 개선

###### 동일한 객체가 요청되는 일이 잦고 객체를 만드는 비용이 크다면 정적 팩터리 메서드를 활용하는 것이 좋다. 이는 경량 패턴과 유사하며, 위의 예제 Boolean.valueOf(Boolean)가 위의 기법을 잘 활용한 형태이다.<br>객체를 반복해서 반환할 수 있으므로 특정 시점에 어떤 객체의 수를 제어할 수 있다. 개체 수를 제어하면 싱글턴 패턴(1개)), 객체 생성이 불가능한 클래스, 변경이 불가능한 클래스(두 개의 같은 객체 존재하지 못하도록)를 만들 수 있다.

{% highlight ruby %}
public static Boolean valueOf(boolean b) {
  return b ? Boolean.TRUE : Boolean.FALSE;
}
{% endhighlight %}

**3. 자료형의 하위 자료형 객체를 반활할 수 있음**
* 반환되는 객체를 유연하게 사용할 수 있음

###### public 으로 선언되지 않은 클래스의 객체를 반환하는 API 생성 가능하고 세부사항을 숨길 수 있다.(인터페이스 기반 프레임워크 구현에 적합). 프레임워크에서 인터페이스는 정적 메서드를 가질 수 없기에 정적 팩터리 메서드의 반환 값 자료형(Types 객체 생성 불가능 클래스에 존재)으로 이용된다.<br>사용자가 자바의 컬렉션 프레임워크 API를 사용할 때 규정된 내용을 읽지않아도 그 구현체를 이용할 수 있다.<br>인자에 따라 객체를 동적으로 결정할 수 있다. enum의 경우 64개 이하의 상수일 경우 RegularEnumSet 객체 반환, 이보다 많으면 jumboEnumSet 객체 반환. 둘다 long Type의 배열을 사용하고 EnumSet의 하위 클래스

###### 정적 팩터리 메서드가 반환하는 객체의 클래스는 정적 팩터리 메서드가 정의된 클래스의 코드가 작성되는 순간에 존재하지 않아도 무방. JDBC와 같은 서비스 제공자 프레임워크를 이루는 것이 유연한 정적 팩터리 메서드들이다. 서비스 제공자 프레임워크는 다양한 서비스 제공자들이 하나의 서비스를 구성하는 시스템이고 클라이언트가 실제 구현된 서비스를 이용할 수 있도록 한다. 

{% highlight ruby %}
//서비스 인터페이스
public interface Service { //서비스의 고유한 메서드 }

//서비스 제공자 인터페이스
public interface Provider { Service newService(); }

//서비스 등록과 접근에 사용되는 객체 생성 불가능 클래스
public class Services {
  private Services() {} //객체 생성 방지(규칙 4);

  //서비스 이름과 서비스 간 대응 관계 보관
  private static final Map<String, Provider> providers =
   new ConcurrentHashMap<String, Provider>();
  public static final String DEFAULT_PROVIDER_NAME = "def_name";

  //제공자 등록 API
  public static void registerDefaultProvider(Provider p) {
    registerProvider(DEFAULT_PROVIDER_NAME, p);
  }
  public static void registerProvider(String name, Provider p) {
    providers.put(name, p);
  }

  //서비스 접근 API
  public static Service newInstance() {
    return newInstance(DEFAULT_PROVIDER_NAME);
  }
  public static Service newInstance(String name) {
    Provider p = Providers.get(name);
    if(p == null)
      throw new IllegalArgumentException("No Provider registered with name : " + name);
    return p.newService();
  }
}

{% endhighlight %}

**4. 형인자 자료형(parameterized type) 개체를 만들 때 유용**
* 자료형 명세를 중복하여 형인자가 늘어나는 복잡한 코드를 간결하게 만들 수 있음 -> 자료형 유추

{% highlight ruby %}
Map<String, List<String>> map = new HashMap<String, List<String>>();

public static <K, V> HashMap<K, V> newInstance() {
  return new HashMap<K, V>;
}
{% endhighlight %}

###### jdk 1.6 까지는 미지원 기능

<br>
# 단점
---------------------
###### 정적 팩터리 메서드를 사용하면 아래와 같은 단점이 존재합니다.

**1. public or protected로 선언된 생성자가 없으므로 하위 클래스 생성 불가**
* 상속(Inheritance) 대신 구성(Composition) 기법을 사용하여 해결 ???

**2. 정적 팩터리 메서드가 다른 정적 메서드와 구분하기 쉽지 않음**
* 사용법을 파악하기가 쉽지 않으므로 네이밍이 중요하다.
* 주석을 사용하여 표시하는 경우도 있음

<b>보통적으로 아래의 네이밍을 사용
* valueOf : 인자로 주어진 값과 같은 값을 갖는 객체를 반환(형변환 메서드))
* of : valueOf의 더 간단한 형태
* getInstacne : 인자에 기술된 객체를 반환하지만, 인자와 같은 값을 갖지 않을 수도 있음.(싱글턴은 Only-One)
* newInstance : getInstance와 같지만 호출할 때마다 다른 객체 반환
* getType : getInstace 동일, 반환될 객체의 클래스와 다른 클래스에 팩터리 메서드가 있을 때 사용. Type은 팩터리 메서드가 반환할 객체의 자료형
* newType : newInstance 동일, 반환될 객체의 클래스와 다른 클래스에 팩터리 메서드가 있을 때 사용. Type은 팩터리 메서드가 반환할 객체의 자료형

<br>
# 결론
---------------------
###### 정적 팩터리 메서드와 public 생성자는 용도를 알아야 하고 장단점을 이해해야 한다. 정적 팩터리 메서드의 장점이 많이 생기는 경우에는 정적 팩터리 메서드를 활용하여 프로그램을 더 가독성, 사용성, 효율성 있게 만들어야 한다.


`Effecitve Java 2/E by 조슈아 블로크 책에 내용을 참고하였습니다.`