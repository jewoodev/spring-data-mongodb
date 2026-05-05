> **원문 저작권:** Copyright © 2008-2025 VMware, Inc. (Broadcom Inc.). All rights reserved.
>
> 본 문서의 사본은 개인적인 용도 및 타인에게 배포하는 용도로 제작할 수 있습니다.
> 단, 사본에 대해 어떠한 수수료도 부과해서는 안 되며, 인쇄물이든 전자적 형태이든
> 배포하는 모든 사본에는 본 저작권 고지가 포함되어야 합니다.
>
> **원문:** <https://docs.spring.io/spring-data/mongodb/reference/mongodb/template-api.html>
>
> 본 문서는 학습 목적의 비공식 한국어 번역이며, 정확한 내용은 원문을 참조하시기 바랍니다.

# Template API

`MongoTemplate`과 `org.springframework.data.mongodb.core` 패키지에 함께 들어 있는 반응형(reactive) 대응 클래는 Spring Data MongoDB가 제공하는 MongoDB 지원의 핵심 클래스이다. 두 클래스 모두 데이터베이스와 상호작용하기 위한 풍부한 기능 세트를 제공하며, 문서 생성·수정·삭제·조회뿐 아니라 도메인 객체와 MongoDB 문서 사이의 매핑까지 한 자리에서 다룬다.

> 한 번 설정된 `MongoTemplate`은 thread-safe 하므로 여러 컴포넌트에서 같은 인스턴스를 공유해 재사용해도 된다.

> 원문: <https://docs.spring.io/spring-data/mongodb/reference/mongodb/template-api.html>

이 페이지는 다음 다섯 가지 주제로 구성되어 있다.

| 주제                                              | 한 줄 요약                                     |
|-------------------------------------------------|--------------------------------------------|
| [Convenience Methods](#convenience-methods)     | 자주 쓰는 작업을 짧고 익숙한 이름으로 호출할 수 있게 해주는 메서드 모음. |
| [Execute Callbacks](#execute-callbacks)         | MongoDB 드라이버 객체를 직접 손에 쥐고 작업해야 할 때 쓰는 탈출구. |
| [Fluent API](#fluent-api)                       | 메서드 체이닝으로 읽기 좋게 표현하는 작업 빌더.                |
| [Exception Translation](#exception-translation) | MongoDB 예외를 Spring의 표준 데이터 액세스 예외 계층으로 변환. |
| [Domain Type Mapping](#domain-type-mapping)     | BSON 문서와 도메인 객체 사이의 변환을 담당하는 매핑 계층.        |

---

## Convenience Methods

`MongoTemplate`은 `MongoOperations` 인터페이스를 구현한다. 이 인터페이스에 정의된 메서드의 이름은 MongoDB 자바 드라이버의 `Collection` API에서 쓰는 이름을 가능한 한 그대로 가져왔다. 드라이버를 다뤄본 적이 있다면 새로 외울 단어가 거의 없도록 하기 위해서다.

드라이버와 다른 점은 다음과 같다.

- 매개변수로 `org.bson.Document`를 채워 넘기는 대신 **도메인 객체** 자체를 넘길 수 있다.
- 쿼리·조건·업데이트 작업을 위한 **유연한 빌더 API**(`Query`, `Criteria`, `Update`)를 함께 제공한다.

자주 쓰이는 메서드의 예시는 다음과 같다.

| 카테고리  | 메서드                                                  |
|-------|------------------------------------------------------|
| 조회    | `find`, `findOne`, `findAndModify`, `findAndReplace` |
| 생성/저장 | `insert`, `save`                                     |
| 수정    | `update`, `updateMulti`                              |
| 삭제    | `remove`                                             |

> `MongoTemplate` 인스턴스의 작업을 코드에서 참조할 때는 구현체인 `MongoTemplate`보다 인터페이스인 `MongoOperations`를 통해 받는 편이 권장된다. 의존 방향이 인터페이스 쪽을 향하면 테스트와 교체가 쉬워진다.

---

## Execute Callbacks

편의 메서드만으로는 부족할 때, 즉 드라이버 API를 직접 호출해야 하는 상황이 있을 수 있다. 이런 경우를 위해 `MongoTemplate`은 `execute(...)` 계열의 콜백 메서드를 열어 둔다. 콜백 안에서는 `MongoCollection` 또는 `MongoDatabase` 객체에 직접 접근할 수 있고, 그 과정에서 발생하는 예외도 자동으로 Spring의 예외 계층으로 변환된다.

| 시그니처                                                                 | 용도                                                    |
|----------------------------------------------------------------------|-------------------------------------------------------|
| `<T> T execute(Class<?> entityClass, CollectionCallback<T> action)`  | 도메인 클래스로부터 컬렉션을 찾아 그 위에서 콜백을 실행한다.                    |
| `<T> T execute(String collectionName, CollectionCallback<T> action)` | 이름으로 지정한 컬렉션 위에서 콜백을 실행한다.                            |
| `<T> T execute(DbCallback<T> action)`                                | `MongoDatabase` 단위에서 콜백을 실행한다.                        |
| `<T> T execute(String collectionName, DbCallback<T> action)`         | 컬렉션 이름과 함께 데이터베이스 콜백을 실행한다.                           |
| `<T> T executeInSession(DbCallback<T> action)`                       | 같은 데이터베이스 커넥션 안에서 콜백을 실행해 read-after-write 일관성을 보장한다. |

다음은 `CollectionCallback`을 사용해 특정 인덱스가 존재하는지 확인하는 예시다.

**Imperative**

```java
boolean hasIndex = template.execute("geolocation", collection ->
    Streamable.of(collection.listIndexes(org.bson.Document.class))
        .stream()
        .map(document -> document.get("name"))
        .anyMatch("location_2d"::equals)
);
```

**Reactive**

```java
Mono<Boolean> hasIndex = template.execute("geolocation", collection ->
    Flux.from(collection.listIndexes(org.bson.Document.class))
        .map(document -> document.get("name"))
        .filterWhen(name -> Mono.just("location_2d".equals(name)))
        .map(it -> Boolean.TRUE)
        .single(Boolean.FALSE)
    ).next();
```

두 예시는 같은 일을 한다. 반응형 코드에서는 `MongoCollection`이 publisher를 돌려주므로 `Flux.from(...)`으로 감싸서 사용한다.

---

## Fluent API

`MongoTemplate`은 컬렉션 생성, 인덱스 관리, CRUD부터 Map-Reduce, 집계(Aggregation)까지 폭넓은 작업을 책임진다. 각 메서드에는 옵션이나 nullable 파라미터를 커버하는 여러 오버로드가 함께 있어, 호출 지점에서 시그니처를 고르기가 번거로울 수 있다.

`FluentMongoOperations`는 가장 자주 쓰는 작업들에 대해 더 좁고 더 읽기 좋은 인터페이스를 제공한다. 진입점은 수행하려는 동작 자체(`insert(...)`, `find(...)`, `update(...)` 등)에서 시작하고, 그 뒤로는 **현재 단계에서 의미 있는 메서드만** 노출되도록 설계되어 있다. 마지막 종료(terminating) 메서드를 호출하는 순간 실제 `MongoOperations` 호출이 일어난다.

**Imperative**

```java
List<Jedi> all = template.query(SWCharacter.class)  ➊
    .inCollection("star-wars")  ➋
    .as(Jedi.class)  ➌
    .matching(query(where("jedi").is(true)))  ➍
    .all();
```

**Reactive**

```java
Flux<Jedi> all = template.query(SWCharacter.class)
    .inCollection("star-wars")
    .as(Jedi.class)
    .matching(query(where("jedi").is(true)))
    .all();
```

1. 쿼리에서 사용할 필드를 매핑하기 위한 도메인 타입.
2. 도메인 타입에서 컬렉션 이름을 결정할 수 없거나 다른 컬렉션을 쓰고 싶을 때 지정한다.
3. 결과를 도메인 타입과 다른 타입(예: 프로젝션 DTO)으로 받고 싶을 때 지정한다.
4. 실제로 적용할 쿼리.

### 종료 메서드

| 메서드        | 의미                         |
|------------|----------------------------|
| `first()`  | 결과 중 첫 번째 한 건만 가져온다.       |
| `one()`    | 단일 결과를 가져온다(여러 건이면 예외).    |
| `all()`    | 모든 결과를 리스트로 가져온다.          |
| `stream()` | 결과를 스트림(또는 `Flux`)으로 가져온다. |

지리 공간 쿼리(geo-spatial)에서는 `GeoResults` / `GeoResult` 형태로 결과를 받을 수 있다.

```java
GeoResults<Jedi> results = template.query(SWCharacter.class)
    .as(Jedi.class)
    .near(alderaan)
    .all();
```

### 결과 후처리: `QueryResultConverter`

`map(...)` 단계에서 BSON 문서와 기본 변환기를 함께 받아 결과를 다른 모양으로 변환할 수 있다.

```java
List<Optional<Jedi>> result = template.query(Person.class)
    .as(Jedi.class)
    .matching(query(where("firstname").is("luke")))
    .map((document, reader) -> Optional.of(reader.get()))
    .all();
```

### 프로젝션 최적화에 대한 주의

`as(...)`로 프로젝션 타입을 지정하면, `MongoTemplate`은 응답에서 **프로젝션 대상 타입에 필요한 필드만** 가져오도록 쿼리를 다듬는다. 단, 다음 조건을 만족해야 한다.

- 원본 `Query`에 별도의 필드 제한(`fields()`)이 걸려 있지 않아야 한다.
- 대상 타입이 **닫힌(closed) 인터페이스 프로젝션** 또는 **DTO 프로젝션**이어야 한다.

> `DBRef`에는 이 프로젝션 최적화를 적용할 수 없다.

---

## Exception Translation

Spring 프레임워크는 다양한 데이터 액세스 기술에 대해 **예외 변환(exception translation)** 기능을 제공한다. JDBC와 JPA에서 시작된 이 패턴을 Spring Data MongoDB는 `org.springframework.dao.support.PersistenceExceptionTranslator`를 구현해 MongoDB까지 확장한다.

이렇게 변환된 예외 계층을 사용하면 다음과 같은 이점이 있다.

- MongoDB의 raw error code에 의존하지 않고도 **이식성 있고 의미가 분명한** 예외 처리 코드를 짤 수 있다.
- 모든 데이터 액세스 예외가 루트인 `DataAccessException`을 상속하므로, **하나의 try-catch 블록**으로 데이터베이스 관련 예외를 모두 잡아낼 수 있다.
- 원본 예외(inner exception)와 메시지는 손실되지 않고 그대로 보존된다.

`MongoExceptionTranslator`가 처리하는 매핑 중 일부는 다음과 같다.

| 원본                                                             | 변환 결과                                |
|----------------------------------------------------------------|--------------------------------------|
| `com.mongodb.Network*` 계열                                      | `DataAccessResourceFailureException` |
| `MongoException` 코드 `1003`, `12001`, `12010`, `12011`, `12012` | `InvalidDataAccessApiUsageException` |

> 모든 MongoDB 드라이버 예외가 `MongoException`을 상속하지는 않는다는 점에 유의해야 한다.

### 커스텀 예외 변환기 등록

예외 변환은 `MongoDatabaseFactory`(또는 그것의 reactive 변형)에 사용자 정의 `MongoExceptionTranslator`를 설정하여 구성할 수 있다. 해당 `MongoClientFactoryBean`에 직접 설정해도 된다.

```java
ConnectionString uri = new ConnectionString("mongodb://username:password@localhost/database");
SimpleMongoClientDatabaseFactory mongoDbFactory = new SimpleMongoClientDatabaseFactory(uri);
mongoDbFactory.setExceptionTranslator(myCustomExceptionTranslator);
```

대표적인 커스터마이징 동기는 **트랜잭션의 일시적 실패** 처리이다. write conflict처럼 재시도하면 성공할 수 있는 작업이 있는데, 이런 경우 MongoDB가 붙여 주는 라벨(예: `TransientTransactionError`)을 보고 예외를 별도 타입으로 감싸 재시도 전략을 적용할 수 있다.

---

## Domain Type Mapping

MongoDB 문서와 도메인 클래스 사이의 변환은 `MongoConverter` 인터페이스 구현체가 담당한다. Spring은 기본 구현으로 `MappingMongoConverter`를 제공하지만, 필요하다면 직접 구현해서 갈아 끼울 수도 있다.

`MappingMongoConverter`는 다음 두 갈래로 동작한다.

1. **메타데이터 기반 매핑**: `@Document`, `@Field`, `@Id` 같은 어노테이션이 붙어 있다면 그 정보를 우선해서 매핑한다.
2. **컨벤션 기반 매핑**: 메타데이터가 없는 객체에 대해서도 ID 필드 추정, 클래스 이름으로부터의 컬렉션 이름 결정 같은 규칙을 적용해서 동작한다.

자세한 매핑 어노테이션과 컨벤션은 [Object Mapping](../object-mapping/object-mapping.md) 문서를 참고하면 된다.

---

## 동기 vs 반응형 API

| 구분        | 동기(Imperative)          | 반응형(Reactive)               |
|-----------|-------------------------|-----------------------------|
| 진입 클래스    | `MongoTemplate`         | `ReactiveMongoTemplate`     |
| 인터페이스     | `MongoOperations`       | `ReactiveMongoOperations`   |
| 반환 타입(단건) | `T`                     | `Mono<T>`                   |
| 반환 타입(다건) | `List<T>` / `Stream<T>` | `Flux<T>`                   |
| 실행 모델     | 블로킹 호출                  | 논블로킹, 백프레셔(backpressure) 기반 |

두 API는 메서드 이름과 인자 모양을 거의 동일하게 맞춰 두었기 때문에, 동기 코드에서 익힌 사용법은 그대로 반응형 코드에 옮길 수 있다. 차이점은 결과를 즉시 받느냐, 구독해서 받느냐 정도이다.
