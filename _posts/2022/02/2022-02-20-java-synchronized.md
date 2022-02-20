---
title: (Java) Synchronized & Volatile 이해
tags: [Java]
style: fill
color: dark
description: Java의 Synchornized & Volatile 를 이해하기
---

## Mehtod vs Object Lock
자바에서 Multi Thread 환경에서 Thread-safe를 제공하기 위해 Synchronzied 키워드를 제공한다. 이를 사용할 때 method 전체에 걸 수도 있고 block lock를 통해 일부 영역에만 걸 수 있다.

HashTable에 put과 ConcurrnetHashMap의 putVal 차이를 확인해보자
```javascript
//HashTable Put
public synchronized V put(K key, V value) {
  // Make sure the value is not null
  if (value == null) {
      throw new NullPointerException();
  }

  // Makes sure the key is not already in the hashtable.
  Entry<?,?> tab[] = table;
  int hash = key.hashCode();
  int index = (hash & 0x7FFFFFFF) % tab.length;
  @SuppressWarnings("unchecked")
  Entry<K,V> entry = (Entry<K,V>)tab[index];
  for(; entry != null ; entry = entry.next) {
      if ((entry.hash == hash) && entry.key.equals(key)) {
          V old = entry.value;
          entry.value = value;
          return old;
      }
  }

  addEntry(hash, key, value, index);
  return null;
}

//ConcurrnetHashMap
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            synchronized (f) {
                /* 생략 */
						}
				}
		}
}
```
HashTable에서 method synchornized를 걸어서 thread-safe를 지원한다. 어떤 쓰레드가 해당 메소드를 사용할 때 부터 락이 걸리고 다른 쓰레드가 진입 시 blocking이 되고 waiting하여 쓰레드가 끝날때 까지 대기한다. **→ put연산이 많아 질수록 대기하는 쓰레드가 많아지고 이는 성능 이슈가 발생할 것이다.**

ConcurrnetHashMap에서는 method영역안에 synchornzied(f) { } 이라는 object 락을 통해 일부영역에서만 한 쓰레드 접근을 허용한다.

**→ 해쉬 충돌이 발생하는 경우에서만 락을 걸가에 다른 작업을 하는 쓰레드들의 blocking이 발생하지 않으므로 성능 이슈가 개선이 된다.**


## Voliatle - 변수의 가시성을 보장
Java에서 변수들은 non-volatile default이다. 특정 시점에 JVM이 Main Memory에서 데이터를 읽어 캐쉬에 저장하거나 캐쉬에 있는 데이터를 메인 메모리에 쓴다.

두 개의 Thread가 있을 때 하나의 변수를 읽고 쓰는 과정에서 캐쉬값을 사용한다면 실제 결과 값과 나와야 하는 결과 값이 달라질 수 있다.

변수에 volatile을 선언하면 Main Memory에 읽고 쓰기를 기본적으로 한다. 두 개의 쓰레드가 하나의 변수를 읽고 쓸때 매번 메인 메모리까지 접근하여 값을 읽고 쓰기에 원하던 결과 값을 제공한다.

**쓰레드 구조 [참조](https://velog.io/@recordsbeat/스레드-도대체-무엇이길래)**
![preview](https://media.vlpt.us/images/recordsbeat/post/3124166f-4866-4c78-ad36-12ab04eb08ac/image.png)
1. PC 레지스터
    1. CPU 명령어를 수행하는데 필요한 정보
    2. stack-base 로 동작
    3. 각 쓰레드별로 JVM Instruction ㅈ수ㅗ를 가짐
2. JVM Stack
    1. 쓰레드의 수행정보를 Frame을 통해 저장
    2. Thread 시작 시 생성되며 method 호출 시 정보는 stack에 쌓이며 종료되는 시점에 사라짐. 매개/지역/임시 변수 그리고 호출 주소를 저장하고 메소드 종료시 메모리 공간 사라짐
3. Native mtehod Stack
    1. Java외의 언어로 작성된 네이티브 코드들을 위한 stack으로 JNI(Java Native Interface), C/C++코드를 수행하기 위한 stack
4. Method Area
    1. 쓰데드 공유하는 메모리 영역
    2. 클래스, 인터페이스, 메소드, 필드, static 변수 등의 바이트 코드 보관
5. heap
    1. 프로그램 상에서 런타임시 동적으로 할당하여 사용하는 영역

## 결론
- 컬렉션 사용법을 익히고 멀티쓰레드 환경에서의 구현 방법을 익혀두자
