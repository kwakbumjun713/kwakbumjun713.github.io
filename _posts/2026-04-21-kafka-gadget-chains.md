---
title : "RCE via Gadget Chain in Kafka"
date : 2026-04-21 08:47:00 +0900
categories: [Research]
tags: [Java, Kafka, Deserialization, RCE]
---
# TL;DR

현재 rewrite lab에서 “Finding vulnerable gadget chains in OpenJDK” 라는 주제로 연구를 진행하고 있다. 해당 연구 진행 중 개인적으로 spring에서 작동하는 rce 가젯 체인을 발견했다. 따라서 이를 활용하여 Java 기반 라이브러리 혹은 프레임워크에서 RCE를 찾아보고자 한다.

나는 이전 zer0con에서 들었던 발표 중 Kafka RCE 관련 발표를 인상 깊게 들었고 Kafka가 Java 기반이기에 여기서 RCE via Gadget chains를 찾아보도록 한다.

# Prerequisites

Kafka에서 Java deserialization gadget chain이 실제로 성립하기 위해서 필요한 조건이 몇 가지 있다.

## 1. readObject() Source

Kafka 어딘가에 바이트스트림을 역직렬화하는 readObject 같은 코드가 있어야 한다.

```java
new ObjectInputStream(input).readObject() 또는 
ObjectInputStream 하위 클래스의 readObject()
```

현재 확인한 Kafka source는 Kafka connect 쪽에 존재하는 아래의 코드가 있다.

```java
@SuppressWarnings("unchecked")
private void load() {
    try (SafeObjectInputStream is = new SafeObjectInputStream(Files.newInputStream(file.toPath()))) {
        Object obj = is.readObject();
        if (!(obj instanceof HashMap))
            throw new ConnectException("Expected HashMap but found " + obj.getClass());
        Map<byte[], byte[]> raw = (Map<byte[], byte[]>) obj;
        data = new HashMap<>();
        for (Map.Entry<byte[], byte[]> mapEntry : raw.entrySet()) {
            ByteBuffer key = (mapEntry.getKey() != null) ? ByteBuffer.wrap(mapEntry.getKey()) : null;
            ByteBuffer value = (mapEntry.getValue() != null) ? ByteBuffer.wrap(mapEntry.getValue()) : null;
            data.put(key, value);
            OffsetUtils.processPartitionKey(mapEntry.getKey(), mapEntry.getValue(), keyConverter, connectorPartitions);
        }
    } catch (NoSuchFileException | EOFException e) {
        // NoSuchFileException: Ignore, may be new.
        // EOFException: Ignore, this means the file was missing or corrupt
    } catch (IOException | ClassNotFoundException e) {
        throw new ConnectException(e);
    }
}
```

→ SafeObjectInputStream으로 바이트스트림을 필터링 중이다. 이 내용은 아래에서 다시 다룬다.

→ 아직 Kafka Broker protocol request path에는 일반적인 ObjectInputStream.readObject() Source가 보이지 않고 있다.

## 2. 공격자가 Source 입력 바이트 제어 가능

가젯 체인은 공격자의 바이트스트림이 `readObject()`에 들어가야 한다. Kafka Connect sink 기준 입력 지점은 `offset.storage.file.filename`이다.

> **조건**
> 공격자가 해당 파일 내용을 쓸 수 있거나, worker가 공격자 제어 파일을 읽도록 경로를 조작할 수 있어야 한다.
> 예를 들면 symbolic link, mount/shared volume, insecure file permissions 같은 조건으로 파일 제어가 가능해야 한다.

## 3. Source가 외부 보안 경계 안에 있어야 함

이 말이 무슨 뜻이냐면 admin 권한으로 바이트스트림을 넣는 것은 취약점이라고 주장하기에는 약한 조건이다. 따라서 공격자는 원격 사용자, 낮은 권한의 connector / operator, REST API 사용자, topic producer 권한으로 serialized bytes를 넣을 수 있어야 한다.

→ 어차피 post exploit에서 활용하기 위해 연구를 하는 것이기에 credential은 탈취 했다고 가정해도 될듯

## 4. 이전에 찾은 spring chain의 한계

내가 찾은 가젯은 spring에서 동작하는 가젯 체인이다. 따라서 타겟의 class path에 spring이 존재해야한다.

→ 본 문제의 해결책으로는 타겟을 spring kafka로 잡으면 될듯함 + 혹은 다른 서드파티 추가

## Deserialization filter 우회 해야함

아까 위의 코드에서 봤던 것처럼 Kafka Connect는 ObjectInputStream이 아니라 

SafeObjectInputStream를 사용중이다. 

```java
<blocklist>

protected static final Set<String> DEFAULT_NO_DESERIALIZE_CLASS_NAMES = Set.of(
        "org.apache.commons.collections.functors.InvokerTransformer",
        "org.apache.commons.collections.functors.InstantiateTransformer",
        "org.apache.commons.collections4.functors.InvokerTransformer",
        "org.apache.commons.collections4.functors.InstantiateTransformer",
        "org.codehaus.groovy.runtime.ConvertedClosure",
        "org.codehaus.groovy.runtime.MethodClosure",
        "org.springframework.beans.factory.ObjectFactory",
        "com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl",
        "org.apache.xalan.xsltc.trax.TemplatesImpl"
);
```

→ 위의 가젯들을 차단하고 있다. 

따라서 우리는 이를 우회하거나, 필터가 없는 readObject를 찾거나, blocklist에 없는 가젯 체인을 사용하여 이를 우회하거나, TemplatesImpl를 Kafka Class로 대체, SafeObjectInputStream을 우회할 수 있는 경로를 찾아야한다. → bypass filter
