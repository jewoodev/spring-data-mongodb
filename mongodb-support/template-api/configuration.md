> **원문 저작권:** Copyright © 2008-2025 VMware, Inc. (Broadcom Inc.). All rights reserved.
>
> **원문 출처:** https://docs.spring.io/spring-data/mongodb/reference/[해당-페이지-경로]
>
> **본 문서는 학습 목적의 비공식 한국어 번역이며, 어떠한 수익도 발생시키지 않습니다.**

# Template Configuration

`MongoTemplate`은 한 번 만들어 두면 thread-safe 하게 재사용할 수 있는 무거운(heavy) 컴포넌트다. 따라서 보통은 Spring 컨테이너에 빈으로 한 번만 등록해 두고, 필요한 곳에서 주입받아 쓴다. 이 문서는 `MongoTemplate`을 어떻게 만들고, 어떤 옵션을 켜거나 끌 수 있는지를 정리한다.

> 원문: <https://docs.spring.io/spring-data/mongodb/reference/mongodb/template-config.html>

| 주제                                                                       | 한 줄 요약                                                       |
|--------------------------------------------------------------------------|--------------------------------------------------------------|
| [Registering the Template](#registering-the-template)                    | `MongoClient`와 `MongoTemplate`을 빈으로 등록하는 가장 기본적인 형태.         |
| [Constructor Overloads](#constructor-overloads)                          | 어떤 인자를 넘기느냐에 따른 생성자 선택지.                                     |
| [WriteResultChecking](#writeresultchecking)                              | 쓰기 결과에 에러가 들어 있을 때 어떻게 반응할지에 대한 정책.                          |
| [WriteConcern](#writeconcern)                                            | 기본 `WriteConcern`과, 작업별로 다르게 적용하기 위한 `WriteConcernResolver`. |
| [ReadPreference](#readpreference)                                        | 읽기 작업에 적용할 기본 read preference.                               |
| [Lifecycle Events / EntityCallbacks](#lifecycle-events--entitycallbacks) | 영속 라이프사이클 이벤트의 on/off, 그리고 콜백 등록.                            |
| [Document Count](#document-count)                                        | 빈 필터의 `count()`를 `estimatedCount()`로 위임할지 여부.                |

---

## Registering the Template

`MongoTemplate`을 등록하려면 보통 두 개의 빈이 필요하다. 하나는 드라이버 수준의 `MongoClient`이고, 다른 하나는 그 위에 올라가는 `MongoTemplate`이다.

**Imperative**

```java
@Configuration
class ApplicationConfiguration {

    @Bean
    MongoClient mongoClient() {
        return MongoClients.create("mongodb://localhost:27017");
    }

    @Bean
    MongoOperations mongoTemplate(MongoClient mongoClient) {
        return new MongoTemplate(mongoClient, "geospatial");
    }
}
```

**Reactive**

```java
@Configuration
class ReactiveApplicationConfiguration {

    @Bean
    MongoClient mongoClient() {
        return MongoClients.create("mongodb://localhost:27017");
    }

    @Bean
    ReactiveMongoOperations mongoTemplate(MongoClient mongoClient) {
        return new ReactiveMongoTemplate(mongoClient, "geospatial");
    }
}
```

**XML**

```xml
<mongo:mongo-client host="localhost" port="27017" />

<bean id="mongoTemplate" class="org.springframework.data.mongodb.core.MongoTemplate">
    <constructor-arg ref="mongoClient" />
    <constructor-arg name="databaseName" value="geospatial" />
</bean>
```

> 빈 타입은 구현체가 아닌 `MongoOperations` / `ReactiveMongoOperations` 인터페이스로 노출하는 편이 좋다. 사용 측이 인터페이스에 의존하면 테스트나 교체가 쉽다.

---

## Constructor Overloads

`MongoTemplate`의 생성자는 인자를 어디까지 받느냐에 따라 세 가지가 있다.

| 시그니처                                                                    | 언제 쓰나                                                               |
|-------------------------------------------------------------------------|---------------------------------------------------------------------|
| `MongoTemplate(MongoClient mongo, String databaseName)`                 | 가장 단순한 형태. 드라이버 클라이언트와 기본 데이터베이스 이름만 넘긴다.                           |
| `MongoTemplate(MongoDatabaseFactory factory)`                           | `MongoClient` + DB 이름 + 자격증명까지 한 덩어리로 캡슐화한 팩토리를 쓰고 싶을 때.            |
| `MongoTemplate(MongoDatabaseFactory factory, MongoConverter converter)` | 위와 같지만, 도메인 ↔ BSON 변환을 책임지는 `MongoConverter`를 직접 지정해 매핑을 커스터마이즈할 때. |

마지막 시그니처는 매핑 규칙을 사용자 정의 `MongoConverter`로 갈아 끼우고 싶을 때 사용한다. 자세한 내용은 [Domain Type Mapping](README.md#domain-type-mapping) 절을 참고하면 된다.

---

## WriteResultChecking

쓰기 작업이 끝나면 드라이버는 `com.mongodb.WriteResult`를 돌려준다. 그 안에 에러가 섞여 있을 때 `MongoTemplate`이 어떻게 반응할지를 `WriteResultChecking` 속성으로 결정할 수 있다.

| 값           | 동작                                  |
|-------------|-------------------------------------|
| `NONE`      | 아무 일도 하지 않는다. **기본값**이다.            |
| `EXCEPTION` | 에러가 있으면 예외를 던진다. 개발 단계에서 빠르게 잡기 좋다. |

운영 환경에서는 보통 기본값 그대로 두지만, 개발/테스트 단계에서는 `EXCEPTION`으로 올려 두면 잘못된 쓰기를 조용히 흘려보내지 않고 즉시 드러낼 수 있다.

---

## WriteConcern

`WriteConcern`은 "어디까지 써졌어야 ‘쓰기 성공’으로 간주할 것인가"에 대한 정책이다. MongoDB 클라이언트나 컬렉션 수준에서 이미 설정되어 있다면 그 값이 우선한다. 그렇지 않은 경우, `MongoTemplate`에 직접 기본 `WriteConcern`을 지정해 둘 수 있다.

### 작업마다 다른 WriteConcern: `WriteConcernResolver`

모든 쓰기에 같은 `WriteConcern`을 쓰는 게 아니라, **엔티티 종류나 작업 종류에 따라** 다르게 가져가고 싶을 때가 있다. 예를 들어 감사(audit) 로그는 `ACKNOWLEDGED`로 가볍게, 메타데이터처럼 손실되면 곤란한 데이터는 `JOURNALED`로 더 강하게 보장하고 싶을 수 있다. 이런 경우에는 `WriteConcernResolver`를 구현해 끼워 넣는다.

```java
public interface WriteConcernResolver {
    WriteConcern resolve(MongoAction action);
}
```

```java
public class MyAppWriteConcernResolver implements WriteConcernResolver {

    @Override
    public WriteConcern resolve(MongoAction action) {
        if (action.getEntityType().getSimpleName().contains("Audit")) {
            return WriteConcern.ACKNOWLEDGED;
        } else if (action.getEntityType().getSimpleName().contains("Metadata")) {
            return WriteConcern.JOURNALED;
        }
        return action.getDefaultWriteConcern();
    }
}
```

`MongoAction`은 결정에 필요한 정보를 한꺼번에 들고 다닌다.

| 정보                     | 설명                                                        |
|------------------------|-----------------------------------------------------------|
| 컬렉션 이름                 | 어떤 컬렉션에 대한 작업인가.                                          |
| POJO `java.lang.Class` | 어떤 도메인 타입인가.                                              |
| 변환된 `Document`         | 실제로 쓰여질 BSON 문서.                                          |
| Operation 타입           | `REMOVE`, `UPDATE`, `INSERT`, `INSERT_LIST`, `SAVE` 중 하나. |
| Default `WriteConcern` | 직접 결정하지 않을 때 돌려보낼 기본값.                                    |

리졸버는 직접 결정해야 하는 경우에만 분기를 만들고, 나머지는 `action.getDefaultWriteConcern()`으로 흘려보내면 된다.

---

## ReadPreference

`MongoTemplate`에는 기본 `ReadPreference`를 지정해 둘 수 있다. 이 값은 개별 `Query`에서 read preference가 따로 명시되지 않았을 때 폴백(fallback)으로 쓰인다. 쿼리별로 더 세밀하게 조정하려면 [Query 단위의 read preference](https://docs.spring.io/spring-data/mongodb/reference/mongodb/template-query-operations.html#mongo.query.read-preference)를 참고하면 된다.

---

## Lifecycle Events / EntityCallbacks

`MongoTemplate`은 저장/조회 시점에 `BeforeSaveEvent` 같은 영속(persistence) 이벤트를 발행한다. 자세한 라이프사이클 단계는 [Lifecycle Events](https://docs.spring.io/spring-data/mongodb/reference/mongodb/lifecycle-events.html) 문서에 정리되어 있다.

### 이벤트 발행 끄기

이벤트 리스너를 쓰지 않거나, 성능상 이벤트 발행 자체가 부담스러울 때는 다음과 같이 꺼 둘 수 있다.

```java
@Bean
MongoOperations mongoTemplate(MongoClient mongoClient) {
    MongoTemplate template = new MongoTemplate(mongoClient, "geospatial");
    template.setEntityLifecycleEventsEnabled(false);
    // ...
    return template;
}
```

### EntityCallbacks 등록

이벤트 대신, 또는 이벤트와 함께 **엔티티 콜백**을 끼워 넣어 도메인 객체가 영속되기 직전·직후에 변형(transform)을 가할 수 있다.

**Imperative**

```java
@Bean
MongoOperations mongoTemplate(MongoClient mongoClient) {
    MongoTemplate template = new MongoTemplate(mongoClient, "...");
    template.setEntityCallbacks(EntityCallbacks.create(...));
    // ...
    return template;
}
```

**Reactive**

```java
@Bean
ReactiveMongoOperations mongoTemplate(MongoClient mongoClient) {
    ReactiveMongoTemplate template = new ReactiveMongoTemplate(mongoClient, "...");
    template.setEntityCallbacks(ReactiveEntityCallbacks.create(...));
    // ...
    return template;
}
```

> 이벤트가 "이 일이 일어났다"를 알리는 알림이라면, `EntityCallback`은 "이 시점에 이 객체를 이렇게 바꿔라"를 표현하는 변환 훅(hook)에 가깝다. 둘은 목적이 다르므로 필요에 따라 함께 써도 된다.

---

## Document Count

`MongoTemplate#useEstimatedCount(true)`로 켜 두면, **빈 필터로 호출되는 `count()`** 가 내부적으로 `estimatedCount()`로 위임된다. `estimatedCount()`는 컬렉션 메타데이터를 보고 빠르게 근사값을 돌려주기 때문에 큰 컬렉션에서 압도적으로 빠르다. 단, 다음 두 조건을 모두 만족해야 자동 위임이 일어난다.

- 활성 트랜잭션이 없어야 한다.
- 템플릿이 [client session](https://docs.spring.io/spring-data/mongodb/reference/mongodb/client-session-transactions.html)에 묶여 있지 않아야 한다.

```java
@Bean
MongoOperations mongoTemplate(MongoClient mongoClient) {
    MongoTemplate template = new MongoTemplate(mongoClient, "geospatial");
    template.useEstimatedCount(true);
    // ...
    return template;
}
```

> "정확한 개수"가 필요한 경우라면 켜지 않는 편이 안전하다. 추정값이 필요한 통계/대시보드 용도에 적합하다. 자세한 비교는 [Counting Documents](https://docs.spring.io/spring-data/mongodb/reference/mongodb/template-document-count.html#mongo.query.count) 문서를 참고하면 된다.
