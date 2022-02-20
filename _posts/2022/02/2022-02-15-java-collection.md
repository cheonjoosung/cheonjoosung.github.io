---
title: (Java) JCF(Java Collection Framework) & Concurrent 구현방법
tags: [Java]
style: fill
color: dark
description: Java의 Collection Framework 인 List, Set, Queue, Map 및 멀티쓰레드 환경에서의 컬렉션 사용방법
---

## JCF (Java Collection Framework)
JCF는 재사용 가능한 콜렉션 데이터 구조를 구현하는 클래스 및 인터페이스 세트입니다. 프레임 워크라고도하지만 라이브러리 방식으로 작동합니다. 컬렉션 프레임 워크는 다양한 컬렉션과이를 구현하는 클래스를 정의하는 인터페이스를 모두 제공합니다. ([JCF 구글 위키](https://en.wikipedia.org/wiki/Java_collections_framework))

![preview](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/4a03909a-f4f6-4b54-869e-077f8d19b69e/Untitled.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45EIPT3X45%2F20220219%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20220219T122705Z&X-Amz-Expires=86400&X-Amz-Signature=b5b4cbaab8eb48149ac4b4aa6285f9e4a653a46d70f8cba7716f789cfc61e61c&X-Amz-SignedHeaders=host&response-content-disposition=filename%20%3D%22Untitled.png%22&x-id=GetObject)

![preview](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/2c13bf58-d422-46a2-879f-f53a032c32d8/Untitled.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45EIPT3X45%2F20220219%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20220219T122711Z&X-Amz-Expires=86400&X-Amz-Signature=2b01a9eea6ef7f8d37dc968efe1be04a0604d80d388a19000530465c6229abca&X-Amz-SignedHeaders=host&response-content-disposition=filename%20%3D%22Untitled.png%22&x-id=GetObject)
책이나 인터넷 검색해보면 추상클래스가 생략되어 있는 그림이 많다. 내부적인 실제 구조는 위의 그림과 같다.

## List
List를 구현한 것이 AbstractList Class 이고 이를 상속하여 만든 클래스에 ArrayList, Vector, Stack이 있습ㄴ다. List는 순서가 있고 중복을 허용하는 자료구조입니다.
```javascript
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable

public class Vector<E>
    extends AbstractList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable

public class Stack<E> extends Vector<E>
```
각각은 AbstractList를 상속받았고  RandomAccess(인덱스 접근), Cloneable(클론), Serializable(직렬화)인터페이스를 구현한다. (스택은 백터를 상속받았으니 거기서 거기...)

List를 구현한 클래스에는 AbstractList, AbstarctequentialLit, **ArrayList**, AttributeList, CopyOnWriteArrayList, **LinkedList**, RoleLit, RoleUnresolvedList, Stack, Vector가 있다

**ArrayList**
배열처럼 인덱스를 활용하여 빠르게 접근이 가능한 자료구조이다. 배열은 고정 사이즈이지만 ArrayList는 동적으로 크기가 변한다는 점이다. 추가/삭제에 대해서 시간이 더 소요될 수 있다.

간단하게 사용법을 알아보자
```javascript
ArrayList<String> arrayList = new ArrayList<>();

// 추가
arrayList.add("1");
arrayList.add("2");
arrayList.add("3");
arrayList.add("4");
arrayList.add("5");

// 추가2
String [] arr = new String[3];
arrayList.addAll(new ArrayList<String>());
Collections.addAll(arrayList, arr);

// 삭제
String a = "4";
arrayList.remove(1);
arrayList.remove(a);
arrayList.clear();

// 변경
arrayList.set(0, "99");

// 검색
a = "3";
arrayList.indexOf(a);
arrayList.indexOf(3);

// 탐색
int size = arrayList.size();
for (int i=0 ; i<size ; i++) System.out.println(arrayList.get(i));
for (String s : arrayList) System.out.println(s);

Iterator<String> it = arrayList.iterator();
while (it.hasNext()) System.out.print(it.next() + " ");
```

**Vector**
Vector Method는 ArrayList Method와 거의 유사하다. Legacy Class로 오래된 자바 버전의 유산이라고 생각하면 되고 현재 컬렉션 프레임워크와 완벽하게 호환된다. 실질적으로 잘 사용되지 않고 대체가능한 클래스인 ArrayList가 존재한다.

ArrayList와의 차이점은 method에 `synchronized를 통한 Thread Safe`이다. 싱글 쓰레드 환경에서 메소드에 접근할 때 락을 걸고 푸는 과정이 발생하므로 성능은 ArrayList가 조금 더 좋다.

**Stack**
Stack은 Vector 클래스를 상속받아 구현했고 LIFO(Last In First Out)으로 나중에 들어간 것이 가장 빨리 나오는 특징을 가진다.

Vector 클래스를 상속받았기에 특이점이 있다.

- stack-pop/push 메소드가 있지만 vector-remove(index)/set(index,value)를 통해 삭제가 가능하다. vector 메소드를 사용이 가능
- thread-safe 하므로 멀티쓰레드 환경이 아닐때 메소드를 호출할 때 마다 오버헤드가 발생한다.

**LinkedList**
LinkedList는 위의 자료구조와 조금 다른 특징을 가진다.
```javascript
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable

    transient Node<E> first;
    transient Node<E> last;

private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```
AbstractSequentialList(AbstractList 상속)클래스를 상속받았고 Deque 인터페이스를 구현하고 있다. LinkedList에서 자주 쓰이는 node의 제일 앞/뒤, 중간 삽입등에 대한 method를 정의하고 있다.

## Set
Set 의 특징은 순서를 관리하지 않고 중복 데이터를 허용하지 않는 특징을 가진다. 이를 구현한 클래스에는 대표적으로 HashSet, TreeSet 이 있다.
```javascript
public class HashSet<E>
    extends AbstractSet<E>
    implements Set<E>, Cloneable, java.io.Serializable

public class TreeSet<E> extends AbstractSet<E>
    implements NavigableSet<E>, Cloneable, java.io.Serializable
```
Set 인터페이스 구현한 클래스 AbstractSet, ConcurrentSkipListSet, CopyOnWriteArraySet, EnumSet, HashSet, JobStateReasons, LinkedHashSet, TreeSet가 있다.

**HashSet**
HashSet은 내부적으로  HashMap을 활용하여 데이터를 관리한다.
```javascript
private transient HashMap<E,Object> map;

// Dummy value to associate with an Object in the backing Map
private static final Object PRESENT = new Object();


// HashSet
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}
public boolean remove(Object o) {
    return map.remove(o)==PRESENT;
}

// HashMap
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
public V remove(Object key) {
    Node<K,V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}
```
HashMap으로 사용하는데 Set 을 사용할 때 Key값만 던지고 Value는 따로 세팅하지 않는다. 내부적으로 Object PRESENT라는 static 변수를 사용하여 value값이 모두 동일한 더미 객체를 가리키고 있다.

→ Set 은 Key값만 관리하면 되는 특징이 있기때문에 상관이 없다.

간단하게 사용법을 알아보자
```javascript
Set<String> hs = new HashSet<>();
Set<String> hs2 = new HashSet<>();

//추가
hs.add("5");
hs.add("6");
hs.add("7");
System.out.println(hs.add("5")); // 중복요소 false return

//탐색
if (hs.contains("4")) System.out.println("find key");
for (String key : hs) System.out.println(key);
hs.forEach(System.out::println);

//삭제
hs.remove("5");
hs.removeAll(hs2); //collection과 같은 아이템 제거
hs.clear();
```
중복요소의 경우 false가 return이 되고 HashSet에 추가되지 않는다.

**TreeSet**
TreeSet은 데이터가 정렬된 상태로 저장되는 자료구조이며 이진검색트리(레드 블랙 트리) 형태로 데이터를 저장.
```javascript
private transient NavigableMap<E,Object> m;

// Dummy value to associate with an Object in the backing Map
private static final Object PRESENT = new Object();
```
HashSet가 동일하게 PRESNET 더미 객체를 사용한다.


간단하게 사용법을 알아보자
```javascript
TreeSet<String> ts = new TreeSet<>();

//추가
ts.add("7");
ts.add("5");
ts.add("6");
System.out.println(ts.add("5")); // 중복요소 false return

//탐색
if (ts.contains("4")) System.out.println("find key");
for (String key : ts) System.out.println(key);
ts.forEach(System.out::println);

//삭제
ts.remove("5");
ts.removeAll(ts);
ts.clear();
```
삽입 7, 5, 6를 해도 정렬이 되므로 탐색을 하면 5, 6, 7로 결과가 나온다.

![preview](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/86a5a219-99ca-471a-967b-5994fdc52b26/Untitled.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45EIPT3X45%2F20220215%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20220215T110618Z&X-Amz-Expires=86400&X-Amz-Signature=30a5c7c3d381cc96acd716a45e7910734c659cb5d68c00fb77d1177108a0866a&X-Amz-SignedHeaders=host&response-content-disposition=filename%20%3D%22Untitled.png%22&x-id=GetObject)
이진검색의 탐색속도는 높이로 결정되기에 루트 노드 삽입 이후에 그보다 큰 값만 있으면 우측으로 편향된 트리가 발생합니다. 탐색시간 O(n) or O(h)

레드블랙트리는 balanced BST로 이러한 문제가 발생하지 않는다. 데이터 추가/삭제에 따라 조건을 맞추기 위해서 rebalancing이 계속해서 일어난다. 참고 :([https://zeddios.tistory.com/237](https://zeddios.tistory.com/237))

특징 (참고 : [https://ko.wikipedia.org/wiki/레드-블랙_트리](https://ko.wikipedia.org/wiki/%EB%A0%88%EB%93%9C-%EB%B8%94%EB%9E%99_%ED%8A%B8%EB%A6%AC))

- 노드는 레드 혹은 블랙 중의 하나이다.
- 루트 노드는 블랙이다.
- 모든 리프 노드들(NIL)은 블랙이다.
- 레드 노드의 자식노드 양쪽은 언제나 모두 블랙이다. (즉, 레드 노드는 연달아 나타날 수 없으며, 블랙 노드만이 레드 노드의 부모 노드가 될 수 있다)
- 어떤 노드로부터 시작되어 그에 속한 하위 리프 노드에 도달하는 모든 경로에는 리프 노드를 제외하면 모두 같은 개수의 블랙 노드가 있다.


## Queue
Queue는 스택과 달리 인터페이스 구조로 되어있으며 이를 구현한 클래스 클래스 AbstractQueue, ArrayBlockingQueue, ArrayDeque, ConcurrentLinkedQueue, DelayQueue, LinkedBlockingDeque, LinkedBlockingQueue, **LinkedList**, PriorityBlockingQueue, PriorityQueue, SynchronousQueue이 있다.

이 중에 LinkedList 관한 내용은 위에서 참조하고 PriorityQueue에 대해 파악을 해봅시다.
```javascript
public interface Queue<E> extends Collection<E>
    boolean add(E e);   // 요소 추가. + 큐 가득찰 때 exception 발생
    boolean offer(E e); // add와 유사 + 큐 가득찰 때 false 리턴 -> 예외처리 필요

    E remove();    // 요소 반환 + 제거 o
    E poll();     // 요소반환 + 비어있으면 null
    E element();  // 요소 반환 + 제거 x
    E peek();    // 요소 반환 + 비어있으면 null
```

**PriorityQueue**
큐에 데이터를 추가/삽입을 할 때 우선순위가 높은 순서대로 정렬이 되는 특징을 가진다
```javascript
public boolean add(E e) return offer(e)
public boolean offer(E e) {
    if (e == null)
        throw new NullPointerException();
    modCount++;
    int i = size;
    if (i >= queue.length)
        grow(i + 1);
    siftUp(i, e);
    size = i + 1;
    return true;
}

private void siftUp(int k, E x) {
    if (comparator != null)
        siftUpUsingComparator(k, x, queue, comparator);
    else
        siftUpComparable(k, x, queue);
}

private static <T> void siftUpUsingComparator(
    int k, T x, Object[] es, Comparator<? super T> cmp) {
    while (k > 0) {
        int parent = (k - 1) >>> 1;
        Object e = es[parent];
        if (cmp.compare(x, (T) e) >= 0)
            break;
        es[k] = e;
        k = parent;
    }
    es[k] = x;
}
```
내부적으로 comparator를 활용하여 값을 비교하면서 배열(큐 멤버변수)을 재구성한다.

## Map
Map 인터페이스는 (key, value)방식을 쌍으로 데이터를 저장하는 방식을 활용한다. 순서를 따로 관리하지는 않고 key의 중복을 허용하지 않는다.
```javascript
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence,
               Constable, ConstantDesc

    @Stable
    private final byte[] value;
```
Map 인터페이스 구현한 클래스 AbstractMap, Attributes, AuthProvider, ConcurrentHashMap, ConcurrentkipListMap, EnumMap, **HashMap, Hashtable**, IdentityHashMap, LinkedHashMap, PrinterStateReasons, Properties, Provider, RenderingHints, SimpleBindings, TabularDataupport, **TreeMap**, UIDefaults, WeakHashMap이 있다.

TreeMap 특이점은 TreeSet에서 언급한 것처럼 데이터를 이진 검색 트리의 형태로 저장하고 있습니다.

간단하게 사용법을 알아보자
```javascript
Map<String, String> map = new HashMap<>();

// 추가
map.put("key", "value");
map.put("key2", "value2");
map.put("key3", "value3");

// 탐색
for (String key: map.keySet()) {
    System.out.println(key + " - " + map.get(key));
}

// 키셋
if (map.containsKey("key")) System.out.println("find key");
if (map.containsValue("value")) System.out.println("find value");

// 삭제
map.remove("key");
map.clear(); ㅓㅈㄴ체 제거

// 변경
map.replace("key2", "newValue2");

//크기
map.size()
```

**HashMap 동작원리 and 충돌회피**
```javascript
public final boolean equals(Object o) {
      if (o == this)
          return true;
      if (o instanceof Map.Entry) {
          Map.Entry<?,?> e = (Map.Entry<?,?>)o;
          if (Objects.equals(key, e.getKey()) &&
              Objects.equals(value, e.getValue()))
              return true;
      }
      return false;
  }

public final int hashCode() {
    return Objects.hashCode(key) ^ Objects.hashCode(value);
}

static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
위의 있는 HashMap(HashSet이 내부적으로 HashMap을 사용)에서 equals(), HascdeCode()의 메소드 이다.

HashMap의 중복처리는 key와 hash(key)를 값이 던짐. key의 해쉬코드 xor unsinged bit operator를 활용하여 중복처리를 하고 있다.

하지만 중복이 발생할 수도 있는데 회피 방법은?

1. Open Addressing
    1. 일정 공간을 할당하고 순차적으로 저장 (배열 처럼)
2. Separate Chaining
    1. 1번의 방법 버킷에 할당하고 충돌이 발생할 떄 체인으로 연결하는 것
3. 자바8 향상버전
    1. LinkedList로 계속 만들면 성능 이슈
    2. 일정 갯수가 넘어가면 트리로 변경하여 관리
    3. 검색은 트리가 유리고 이떄 redblackTree개념을 활용. [ O(n) vs O(log(n) = h) ]
    4. 대소 관계를 위해 아래의 코드를 활용

```javascript
static int tieBreakOrder(Object a, Object b) {
    int d;
    if (a == null || b == null ||
        (d = a.getClass().getName().
         compareTo(b.getClass().getName())) == 0)
        d = (System.identityHashCode(a) <= System.identityHashCode(b) ?
             -1 : 1);
    return d;
}
```


## 멀티쓰레드 환경에서 콜렉션 사용
위에서 언급한 Collection 중 Vector, Hashtable 은 동기화(syncrhonized)가 적용되어 있어서 Thread Safe가 가능하다. Multi Thread 환경에서 Collection를 사용하기 위한 방법이 있다.

**Collections.synchronizedXXX 사용**
![preview](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/cf333067-bcdd-4aa3-96dc-fc435bd7842d/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2022-02-14_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_1.07.11.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45EIPT3X45%2F20220215%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20220215T105010Z&X-Amz-Expires=86400&X-Amz-Signature=94a970c389751c79d7608e91a08fbd65c808b1cf556dd609e4dba3eb050d3ddb&X-Amz-SignedHeaders=host&response-content-disposition=filename%20%3D%22%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA%25202022-02-14%2520%25E1%2584%258B%25E1%2585%25A9%25E1%2584%2592%25E1%2585%25AE%25201.07.11.png%22&x-id=GetObject)
Collections의 static method로 구현이 되어있어서 바로 접근하여 사용할 수 있다. Collection을 구현한 SynchronizedCollection<E> 과 Map을 구현한 SynchronizedMap<E>을 통해 Thread Safe가 가능한 형태로 변환해 준다.

```javascript
@SuppressWarnings("serial") // Conditionally serializable
final Object mutex // Object on which to synchronize

public V method() {
	synchronized (mutex) {
		// 코드
	}
}
```
위에서 언급한 클래스에서는 mutex라는 변수를 가지고 mutex를 활용하여 동기화 기능을 제공한다.

컴퓨터 과학에서 락(lock) 또는 뮤텍스(mutex, 상호 배제에서)는 여러 스레드를 실행하는 환경에서 자원에 대한 접근에 제한을 강제하기 위한 동기화 매커니즘이다. 락은 상호 배제 동시성 제어 정책을 강제하기 위해 설계된다. [컴퓨터과학](https://ko.wikipedia.org/wiki/락_)

**java.util.concurrent.xxxx 사용**
![preview](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/86a5a219-99ca-471a-967b-5994fdc52b26/Untitled.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45EIPT3X45%2F20220219%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20220219T122746Z&X-Amz-Expires=86400&X-Amz-Signature=cd44fffd8f2b525c33d21e7fd36ec2ecdd1c45208af4f68411b8f98f44731d1c&X-Amz-SignedHeaders=host&response-content-disposition=filename%20%3D%22Untitled.png%22&x-id=GetObject)

위에서는 synchronized & mutex를 활용하여 공유 데이타에 대한 락을 제공했다면 ConcurrentXXX는 다른 방식을 사용한다.
```javascript
public class ConcurrentHashMap<K,V> extends AbstractMap<K,V>
    implements ConcurrentMap<K,V>, Serializable

transient volatile Node<K,V>[] table;

private transient volatile Node<K,V>[] nextTable;

private transient volatile long baseCount;

private transient volatile int sizeCtl;

private transient volatile int transferIndex;

private transient volatile int cellsBusy;

private transient volatile CounterCell[] counterCells;
```
대표적으로 ConcurrentHashMap의 필드변수들을 보면 volatile 키워드를 사용한다. 이 키워드는 변수를 메인 메모리에 저장한다는 의미이다. volatitle을 선언하면 메인 메모리에도 변수를 저장하고 값을 읽어올때 메인 메모리를 참조한다.

![preview](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/b133357b-3590-448a-ab68-a414f714920c/Untitled.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45EIPT3X45%2F20220215%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20220215T105740Z&X-Amz-Expires=86400&X-Amz-Signature=07a399191611e1b4b36e9cf98e099011cc97b797f083b0103c87dca5272d37c1&X-Amz-SignedHeaders=host&response-content-disposition=filename%20%3D%22Untitled.png%22&x-id=GetObject)
**공유자원에 대한 값을** **read/write할 때에는 원자성을 보장해야 하기에 synchronized를 통해 읽고/쓰는 작업 도중에 다른 쓰레드의 읽기 작업을 막아야** 한다.

→ 모든 쓰레드가 쓰기작업이 있는 프로그래밍보다 일부 쓰레드가 쓰기 작업을 하고 나머지는 읽기 작업을 하는 프로그래밍 환경에 잘 어울릴 것이다.

→ **메인 메모리에 매번 읽고/쓰는 작업 됨 & 쓰레드의 캐쉬보다 메인 메모리에 비용이 크기에 성능 고려 필요**


**CopyOnWriteArrayList - ArrayList의 Thread-Safe 지원 버전**
ArrayList는 위에서 언급한 것 처럼 Thread-Safe 하지 않다. 이를 위한 대안은 **CopyOnWriteArrayList 이다.**
```javascript
public class CopyOnWriteArrayList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable
    private static final long serialVersionUID = 8673264195747942595L;

    /**
     * The lock protecting all mutators.  (We have a mild preference
     * for builtin monitors over ReentrantLock when either will do.)
     */
    final transient Object lock = new Object();

    /** The array, accessed only via getArray/setArray. */
    private transient volatile Object[] array;
```
Collections.synchronizedMethod 나 Vector - Synchronized만 사용했다면 위의 클래스는 volatile 키워드를 사용하여 멤버 필드를 관리한다.
```javascript
// ArrayList
public E set(int index, E element) {
    Objects.checkIndex(index, size);
    E oldValue = elementData(index);
    elementData[index] = element;
    return oldValue;
}

public boolean add(E e) {
    modCount++;
    add(e, elementData, size);
    return true;
}

// CopyOnWriteArrayList
public E set(int index, E element) {
    synchronized (lock) {
        Object[] es = getArray();
        E oldValue = elementAt(es, index);

        if (oldValue != element) {
            es = es.clone();
            es[index] = element;
        }
        // Ensure volatile write semantics even when oldvalue == element
        setArray(es);
        return oldValue;
    }
}


public boolean add(E e) {
    synchronized (lock) {
        Object[] es = getArray();
        int len = es.length;
        es = Arrays.copyOf(es, len + 1);
        es[len] = e;
        setArray(es);
        return true;
    }
}
```
CopyOnWriteArrayList 는 말 그대로 변경 작업에 있어서 복사본을 만든다. 복사본에 값 추가/변경을 처리하고 난 후에 쓰기 작업을 진행한다. (Copy On Write)

또한, 읽기 작업(get)에 Lock이 존재하지 않기에 다수의 쓰레드가 읽기 작업을 해도 크게 문제가 없다. Collections.SynchronizedMethod는 한 쓰레드만 공유 자원에 접근가능하므로 읽기 작업에 있어서 성능은 CopyOnWrite가 좋을 수 밖에 없다.

그렇지만 쓰기 작업의 경우 배열 복사가 일어나고 이는 비용이 더 발생한다는 의미이다.


## 결론
- 컬렉션 사용법을 익히고 멀티쓰레드 환경에서의 구현 방법을 익혀두자
