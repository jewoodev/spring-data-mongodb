> **원문 저작권:** Copyright © 2008-2025 VMware, Inc. (Broadcom Inc.). All rights reserved.
>
> **본 문서는 학습 목적의 비공식 한국어 번역이며, 어떠한 수익도 발생시키지 않습니다.**

# Querying Documents

Spring Data MongoDB는 쿼리를 두 개의 작은 빌더 클래스로 표현한다. **`Query`** 는 "어떤 옵션을 켜고 어떤 조건들을 묶어서 보낼 것인가"를, **`Criteria`** 는 "각 필드에 어떤 조건을 걸 것인가"를 책임진다. 메서드 이름은 가능한 한 MongoDB 연산자(`$lt`, `$gte`, `$in` 등)와 같은 모양으로 맞춰져 있어, MongoDB 셸에서 쿼리를 짜본 적이 있다면 거의 그대로 옮겨 쓸 수 있다.

> 원문: <https://docs.spring.io/spring-data/mongodb/reference/mongodb/template-query-operations.html>

| 주제                                            | 한 줄 요약                                          |
|-----------------------------------------------|-------------------------------------------------|
| [Query와 Criteria의 기본기](#query와-criteria의-기본기) | 두 클래스를 어떻게 조립해 쿼리 한 건을 만들어내는가.                  |
| [컬렉션 조회하기](#컬렉션-조회하기)                         | `find`, `findOne`, `findById`와 fluent 진입점의 사용법. |
| [Criteria의 메서드들](#criteria의-메서드들)             | 비교·집합·논리·정규식·지리 연산자에 대응하는 자바 메서드 목록.            |
| [Query의 메서드들](#query의-메서드들)                   | 정렬·페이징·필드 제한 같은 쿼리 옵션을 거는 방법.                   |
| [필드 프로젝션](#필드-프로젝션)                           | 응답에서 필요한 필드만 가져오기, 그리고 표현식으로 가공해서 받기.           |
| [그 밖의 쿼리 옵션](#그-밖의-쿼리-옵션)                     | 힌트, 배치 크기, collation, read preference, 주석 등.    |
| [Distinct 값 조회](#distinct-값-조회)               | 한 필드의 중복 없는 값들만 모아서 받기.                         |
| [지리 공간 쿼리](#지리-공간-쿼리)                         | `$near`, `$within` 등 위치 기반 조회.                  |
| [Geo-near 쿼리](#geo-near-쿼리)                   | 거리까지 함께 받아오는 `$geoNear` 집계 기반 조회.               |
| [GeoJSON 지원](#geojson-지원)                     | GeoJSON 포맷 사용 시 좌표 순서와 거리 단위가 어떻게 달라지는지.        |
| [전문 검색(Full-text)](#전문-검색full-text)           | `$text` 연산자와 `TextCriteria` / `TextQuery`.      |
| [Query by Example](#query-by-example)         | 도메인 객체 한 건을 "샘플"로 넘겨 비슷한 문서를 찾기.                |
| [JSON 스키마로 조회](#json-스키마로-조회)                 | 스키마 구조를 만족하는 문서만 골라오기.                          |

---

## Query와 Criteria의 기본기

`Query`는 쿼리 한 건의 **외피**다. 어떤 조건을 보낼지(=`Criteria`), 어떤 필드를 받을지, 정렬·페이징·힌트 같은 옵션은 무엇인지 같은 정보를 한 객체에 담는다. `Criteria`는 그 안에서 실제 조건식을 만들어내는 빌더이고, 정적 팩토리 메서드 `where(...)`를 진입점으로 삼는다.

가독성을 위해 다음 두 정적 임포트는 거의 항상 함께 쓴다.

```java
import static org.springframework.data.mongodb.core.query.Criteria.where;
import static org.springframework.data.mongodb.core.query.Query.query;
```

이렇게 해두면 쿼리 표현이 영문 문장처럼 읽힌다.

**Imperative**

```java
List<Person> result = template.query(Person.class)
    .matching(query(where("age").lt(50).and("accounts.balance").gt(1000.00d)))
    .all();
```

**Reactive**

```java
Flux<Person> result = template.query(Person.class)
    .matching(query(where("age").lt(50).and("accounts.balance").gt(1000.00d)))
    .all();
```

빌더가 거추장스럽거나, 이미 손에 BSON/JSON 모양의 쿼리가 있는 상황이라면 `BasicQuery`로 문자열 그대로 만들어 쓸 수도 있다.

```java
BasicQuery query = new BasicQuery("{ age : { $lt : 50 }, accounts.balance : { $gt : 1000.00 } }");
List<Person> result = mongoTemplate.find(query, Person.class);
```

---

## 컬렉션 조회하기

조회 결과는 도메인 객체로 매핑되어 돌아온다. 동기 API는 `List<T>` 또는 `T`를, 반응형 API는 `Flux<T>` 또는 `Mono<T>`를 돌려준다.

| 메서드                             | 의미                                    |
|---------------------------------|---------------------------------------|
| `findOne(Query, Class<T>)`      | 조건에 맞는 첫 한 건. 없으면 `null` / 빈 Mono.    |
| `findById(Object, Class<T>)`    | id 기반 단건 조회. 가장 자주 쓰는 단축 경로.          |
| `find(Query, Class<T>)`         | 조건에 맞는 모두를 가져온다.                      |
| `findAll(Class<T>)`             | 컬렉션 전체.                               |
| `query(Class<T>).matching(...)` | fluent 진입점. 위 메서드들과 같은 일을 더 읽기 좋게 표현. |

가장 자주 쓰는 형태는 fluent API의 `template.query(...).matching(...).all()` 흐름이다. 이 방식이 어떤 메서드를 어느 단계에서 부를 수 있는지 IDE가 안내해 주기 때문에 시그니처를 외우는 부담이 적다.

---

## Criteria의 메서드들

`Criteria`의 메서드는 대체로 MongoDB 쿼리 연산자와 1:1 대응한다. 카테고리별로 정리하면 다음과 같다.

### 비교 연산자

| 메서드              | 대응 연산자      | 설명                          |
|------------------|-------------|-----------------------------|
| `is(Object)`     | `{ k : v }` | 정확히 같은 값. 가장 흔한 형태.         |
| `ne(Object)`     | `$ne`       | 같지 않다.                      |
| `lt(Object)`     | `$lt`       | 미만.                         |
| `lte(Object)`    | `$lte`      | 이하.                         |
| `gt(Object)`     | `$gt`       | 초과.                         |
| `gte(Object)`    | `$gte`      | 이상.                         |
| `all(Object...)` | `$all`      | 배열 필드가 주어진 값들을 모두 포함하는지 검사. |

### 집합/배열 연산자

| 메서드                       | 대응 연산자       | 설명                          |
|---------------------------|--------------|-----------------------------|
| `in(Object...)`           | `$in`        | 가변 인자로 넘긴 값들 중 하나에 매칭.      |
| `in(Collection<?>)`       | `$in`        | 컬렉션 버전.                     |
| `nin(Object...)`          | `$nin`       | 어디에도 매칭되지 않을 때.             |
| `size(int)`               | `$size`      | 배열 길이가 정확히 일치할 때.           |
| `elemMatch(Criteria)`     | `$elemMatch` | 배열의 어느 원소가 하위 조건을 만족할 때.    |

### 논리 연산자

| 메서드                        | 대응 연산자 | 설명                               |
|----------------------------|--------|----------------------------------|
| `and(String key)`          | 조건 추가  | 같은 `Criteria`에 다른 필드의 조건을 이어 붙임. |
| `andOperator(Criteria...)` | `$and` | 여러 조건을 명시적으로 AND로 묶어 보낸다.        |
| `orOperator(Criteria...)`  | `$or`  | OR.                              |
| `norOperator(Criteria...)` | `$nor` | 어느 것도 만족하지 않을 때.                 |
| `not()`                    | `$not` | 직전 조건을 부정한다.                     |

`and(...)`는 같은 도큐먼트 내 다른 필드 조건을 이어 붙이는 **체이닝용**이고, `andOperator(...)`는 별도의 `Criteria` 여러 개를 명시적으로 `$and`로 묶을 때 쓴다. 보통은 체이닝만으로 충분하지만, 같은 필드에 두 조건을 동시에 거는 경우처럼 모호함이 생기면 `andOperator(...)`로 명시하는 편이 안전하다.

### 그 밖의 연산자

| 메서드                                          | 대응 연산자        | 설명                       |
|----------------------------------------------|---------------|--------------------------|
| `exists(boolean)`                            | `$exists`     | 필드의 존재 여부 검사.            |
| `regex(String)`                              | `$regex`      | 정규식 매칭.                  |
| `mod(Number, Number)`                        | `$mod`        | 나머지 연산.                  |
| `type(int)`                                  | `$type`       | BSON 타입으로 검사.            |
| `sampleRate(double)`                         | `$sampleRate` | 샘플링 비율로 무작위 추출.          |
| `bits()`                                     | 비트 연산자 진입     | `$bitsAllClear` 등으로 이어짐. |
| `matchingDocumentStructure(MongoJsonSchema)` | `$jsonSchema` | JSON 스키마와 일치하는 문서.       |

### 지리 공간 연산자

| 메서드                                           | 대응 연산자                          | 설명                 |
|-----------------------------------------------|---------------------------------|--------------------|
| `within(Circle)` / `within(Box)`              | `$geoWithin` `$center` / `$box` | 지정한 도형 내부에 들어오는 점. |
| `withinSphere(Circle)`                        | `$geoWithin` `$centerSphere`    | 구면 좌표계 기준의 원 내부.   |
| `near(Point)`                                 | `$near`                         | 가까운 순으로 정렬해서 가져온다. |
| `nearSphere(Point)`                           | `$nearSphere`                   | 구면 거리 기준의 `$near`. |
| `minDistance(double)` / `maxDistance(double)` | `$minDistance` / `$maxDistance` | 거리 범위를 지정.         |

---

## Query의 메서드들

`Query`는 조건 외에 **결과 자체를 어떻게 받을 것인가** 를 다룬다.

| 메서드                              | 의미                                                  |
|----------------------------------|-----------------------------------------------------|
| `addCriteria(Criteria)`          | 조건을 추가로 끼워 넣는다(여러 번 호출 가능).                         |
| `fields()`                       | 결과에 포함하거나 제외할 필드를 지정한다(프로젝션).                       |
| `limit(int)`                     | 최대 결과 개수.                                           |
| `skip(int)`                      | 건너뛸 개수(오프셋 페이징).                                    |
| `with(Sort)`                     | 정렬 기준.                                              |
| `with(ScrollPosition)`           | offset 또는 keyset 기반의 스크롤 페이징.                       |

`fields()`는 다음 절에서 따로 다루고, 페이징과 정렬은 fluent API에서 종료 메서드(`first`, `one`, `all`, `stream`)와 함께 자연스럽게 묶인다.

---

## 필드 프로젝션

응답에 모든 필드가 다 필요한 경우는 의외로 드물다. 큰 문서에서 일부 필드만 꺼내 쓰면 네트워크 비용과 매핑 비용을 동시에 줄일 수 있다.

다음과 같은 도메인이 있다고 하자.

```java
public class Person {
    @Id String id;
    String firstname;

    @Field("last_name")
    String lastname;

    Address address;
}
```

가장 단순한 패턴은 `include`/`exclude`이다.

```java
query.fields().include("lastname");                 // { "_id" : 1, "last_name" : 1 }
query.fields().exclude("id").include("lastname");   // { "_id" : 0, "last_name" : 1 }
query.fields().include("address");                  // 객체 통째로
query.fields().include("address.city");             // 점 표기로 내부 필드 한 개만
```

여기서 한 가지 짚어 둘 점은, **자바 쪽에서는 도메인 필드명**(`lastname`)을 그대로 쓴다는 것이다. 매핑 계층이 알아서 `last_name`으로 바꿔 BSON에 반영해 준다.

### 표현식으로 가공해서 받기 (MongoDB 4.4+)

`fields().project(...).as(...)` 형태를 쓰면, 응답을 받을 때 값을 가공해 새로운 필드 이름으로 받을 수 있다.

```java
// 1) 네이티브 expression - 데이터베이스 필드명을 그대로 사용한다
query.fields()
    .project(MongoExpression.create("'$toUpper' : '$last_name'"))
    .as("last_name");

// 2) AggregationExpression - 도메인 필드명 사용
query.fields()
    .project(StringOperators.valueOf("lastname").toUpper())
    .as("last_name");

// 3) SpEL 표현
query.fields()
    .project(AggregationSpELExpression.expressionOf("toUpper(lastname)"))
    .as("last_name");
```

> 리포지토리 메서드라면 `@Query(fields = "...")` 어트리뷰트로 같은 효과를 얻을 수 있다.

---

## 그 밖의 쿼리 옵션

쿼리에는 조건/프로젝션 외에도 실행 옵션을 여러 가지 붙일 수 있다.

### 인덱스 힌트

옵티마이저가 뽑은 실행 계획이 마음에 들지 않을 때 인덱스를 직접 지정한다.

```java
template.query(Person.class)
    .matching(query(where("...").is("...")).withHint("idx_name"));

// 인덱스 정의를 그대로 넘겨도 된다
template.query(Person.class)
    .matching(query(where("...").is("...")).withHint("{ firstname : 1 }"));
```

### 커서 배치 크기

`cursorBatchSize`는 한 번 왕복할 때마다 가져올 도큐먼트 수를 정한다. 큰 결과 집합을 스트리밍할 때 메모리/네트워크 트레이드오프를 조정하는 손잡이다.

```java
Query query = query(where("firstname").is("luke"))
    .cursorBatchSize(100);
```

### Collation

언어별 정렬·대소문자·악센트(diacritic) 처리 규칙을 따로 줄 수 있다.

```java
Collation collation = Collation.of("de");

Query query = new Query(Criteria.where("firstName").is("Amél"))
    .collation(collation);

List<Person> results = template.find(query, Person.class);
```

### Read Preference

쿼리 단위로 어떤 노드에서 읽을지 지정한다. 여기서 지정한 값은 `MongoTemplate`에 걸린 기본값을 덮어쓴다.

```java
template.find(Person.class)
    .matching(query(where("...").is("...")).withReadPreference(ReadPreference.secondary()))
    .all();
```

### 주석(Comment)

운영 중 슬로우 쿼리 로그나 `currentOp` 출력에서 쉽게 알아보기 위해 주석을 붙여둘 수 있다.

```java
template.find(Person.class)
    .matching(query(where("...").is("...")).comment("Use the force luke!"))
    .all();
```

---

## Distinct 값 조회

특정 필드의 중복 없는 값 목록만 받고 싶을 때 `distinct(...)`를 쓴다.

```java
// 타입 정보 없이 받기 - List<Object>
template.query(Person.class)
    .distinct("lastname")
    .all();

// 타입을 지정해서 받기 - List<String>
template.query(Person.class)
    .distinct("lastname")
    .as(String.class)
    .all();
```

`distinct(...)`에 넘기는 필드 이름은 도메인 필드명이며, `@Field`로 별칭이 걸려 있다면 매핑 계층이 알아서 처리한다.

---

## 지리 공간 쿼리

위치 기반 조회는 `Criteria`의 지리 메서드들과 좌표 도형 클래스(`Circle`, `Box`, `Point`)의 조합으로 표현한다. 다음 도메인을 기준으로 보자.

```java
@Document(collection = "newyork")
public class Venue {

    @Id
    private String id;
    private String name;
    private double[] location;

    Venue(String name, double[] location) {
        this.name = name;
        this.location = location;
    }

    public Venue(String name, double x, double y) {
        this.name = name;
        this.location = new double[] { x, y };
    }
}
```

원·사각형·구 안쪽 검색은 다음과 같다.

```java
// 원 내부
Circle circle = new Circle(-73.99171, 40.738868, 0.01);
List<Venue> venues = template.find(
    new Query(Criteria.where("location").within(circle)), Venue.class);

// 구 내부 (구면 거리 기준)
Circle sphere = new Circle(-73.99171, 40.738868, 0.003712240453784);
List<Venue> venues = template.find(
    new Query(Criteria.where("location").withinSphere(sphere)), Venue.class);

// 사각형 내부 - 좌하단, 우상단 순
Box box = new Box(new Point(-73.99756, 40.73083), new Point(-73.988135, 40.741404));
List<Venue> venues = template.find(
    new Query(Criteria.where("location").within(box)), Venue.class);
```

가까운 순으로 가져올 때는 `near`/`nearSphere`를 쓴다. 거리 범위는 `minDistance`/`maxDistance`로 좁혀준다.

```java
Point point = new Point(-73.99171, 40.738868);

// 평면 거리 기준
List<Venue> venues = template.find(
    new Query(Criteria.where("location").near(point).maxDistance(0.01)), Venue.class);

// 최소·최대 거리 모두 지정
List<Venue> venues = template.find(
    new Query(Criteria.where("location").near(point).minDistance(0.01).maxDistance(100)),
    Venue.class);

// 구면 거리 기준
List<Venue> venues = template.find(
    new Query(Criteria.where("location").nearSphere(point).maxDistance(0.003712240453784)),
    Venue.class);
```

> 트랜잭션 안에서의 지리 쿼리는 일부 제약이 있으므로 [client session / transactions 문서](https://docs.spring.io/spring-data/mongodb/reference/mongodb/client-session-transactions.html)를 함께 참고한다.

---

## Geo-near 쿼리

가까운 순으로 가져오면서 **계산된 거리값** 까지 함께 받고 싶다면 geo-near 형태를 쓴다. MongoDB 4.2부터 `geoNear` 명령은 제거되었고, Spring Data는 내부적으로 `$geoNear` 집계 단계를 사용한다.

```java
Point location = new Point(-73.99171, 40.738868);
NearQuery query = NearQuery.near(location).maxDistance(new Distance(10, Metrics.MILES));

GeoResults<Restaurant> results = operations.geoNear(query, Restaurant.class);
```

- `NearQuery`는 거리·정렬·개수 같은 옵션을 빌더 형태로 묶는다.
- `Metrics.MILES` / `Metrics.KILOMETERS`처럼 사전 정의된 단위를 쓰면 자동으로 spherical 플래그가 설정된다.
- 결과는 평균 거리까지 담은 `GeoResults`이며, 그 안의 각 `GeoResult<T>`가 도메인 객체와 그 객체까지의 거리를 함께 들고 있다.

거리 필드는 응답 도큐먼트에 함께 박혀 들어온다. 만약 도메인 클래스가 같은 이름의 필드를 이미 가지고 있다면, 충돌을 피하기 위해 임의 접미사가 붙은 `calculated-distance` 같은 이름으로 들어가니 주의한다. 거리 필드를 받을 별도 타입을 만들어 `as(...)`로 프로젝션하는 편이 깔끔하다.

```java
GeoResults<VenueWithDistanceField> results = template.query(Venue.class)
    .as(VenueWithDistanceField.class)
    .near(NearQuery.near(new GeoJsonPoint(-73.99, 40.73), KILOMETERS))
    .all();
```

---

## GeoJSON 지원

MongoDB는 위치 데이터를 표현하는 두 가지 포맷을 모두 지원한다. 하나는 `[x, y]` 형태의 **legacy** 좌표 배열, 다른 하나는 `{ type: "Point", coordinates: [x, y] }` 형태의 **GeoJSON** 이다. Spring Data에서는 `org.springframework.data.mongodb.core.geo` 패키지의 `GeoJsonPoint`, `GeoJsonPolygon` 같은 타입을 쓰면 자동으로 GeoJSON 모드로 동작한다.

```java
public class Store {
    String id;

    /** 저장 형태: { "type" : "Point", "coordinates" : [ x, y ] } */
    GeoJsonPoint location;
}
```

> 좌표 순서는 **경도(longitude)가 먼저, 위도(latitude)가 나중** 이다. `GeoJsonPoint.getX()`가 경도, `getY()`가 위도다. 위·경도를 거꾸로 넣는 건 가장 흔한 실수 중 하나이므로 도메인 매핑 시점에 한 번 점검해 두면 좋다.

리포지토리 메서드 시그니처에서 GeoJSON 타입을 쓰면 자동으로 `$geometry` 연산자가 사용된다. 평범한 `Polygon`을 쓰면 legacy `$polygon` 연산자가 만들어진다.

```java
public interface StoreRepository extends CrudRepository<Store, String> {
    List<Store> findByLocationWithin(Polygon polygon);
}

// GeoJSON: $geometry 사용. 첫 점과 마지막 점이 같은 닫힌 링이어야 한다
repo.findByLocationWithin(
    new GeoJsonPolygon(
        new Point(-73.992514, 40.758934),
        new Point(-73.961138, 40.760348),
        new Point(-73.991658, 40.730006),
        new Point(-73.992514, 40.758934)));

// Legacy: $polygon 사용
repo.findByLocationWithin(
    new Polygon(
        new Point(-73.992514, 40.758934),
        new Point(-73.961138, 40.760348),
        new Point(-73.991658, 40.730006)));
```

### 거리 단위가 달라진다

두 포맷의 가장 중요한 차이는 **거리 단위**이다.

| 좌표 포맷    | 거리 단위              | maxDistance 의미   |
|----------|--------------------|------------------|
| Legacy   | 라디안(Radians)       | 지구 반지름 기반의 각거리.  |
| GeoJSON  | 미터(Meters)         | 그대로 미터 단위로 해석된다. |

같은 의미의 쿼리를 둘 모두로 보내면 다음처럼 형태가 달라진다.

```java
// Legacy
NearQuery.near(new Point(-73.99171, 40.738868));
// → "near": [-73.99171, 40.738868]

// GeoJSON
NearQuery.near(new GeoJsonPoint(-73.99171, 40.738868));
// → "near": { "type": "Point", "coordinates": [-73.99171, 40.738868] }
```

저장된 도큐먼트가 GeoJSON 포맷이라면, geo-near 쿼리에서 `maxDistance: 400`은 그대로 400미터로 해석된다. 반면 legacy 포맷에서는 같은 400미터를 표현하려면 라디안으로 환산하고, 결과 거리도 km(`distanceMultiplier: 6378.137`)로 곱해 다시 미터로 만들어야 한다. 새로 만드는 컬렉션이라면 GeoJSON을 쓰는 편이 직관적이고 실수도 적다.

---

## 전문 검색(Full-text)

MongoDB 2.6부터 `$text` 연산자로 전문 검색을 할 수 있다. 단, 검색 대상 필드에 미리 **text 인덱스**를 만들어 둬야 한다.

```javascript
db.foo.createIndex(
    { title : "text", content : "text" },
    { weights : { title : 3 } }
);
```

자바 쪽에서는 `TextCriteria`로 검색어를, `TextQuery`로 쿼리 자체를 표현한다.

```java
Query query = TextQuery.queryText(
    new TextCriteria().matchingAny("coffee", "cake"));

List<Document> page = template.find(query, Document.class);
```

자주 쓰는 옵션은 다음과 같다.

```java
// 관련도(score) 기준 정렬과 점수 포함
Query query = TextQuery.queryText(
        new TextCriteria().matchingAny("coffee", "cake"))
    .sortByScore()
    .includeScore();

// 특정 단어 제외 - 두 가지 방식 모두 동일하게 동작한다
TextQuery.queryText(new TextCriteria().matching("coffee").matching("-cake"));
TextQuery.queryText(new TextCriteria().matching("coffee").notMatching("cake"));

// 구문(phrase) 검색
TextQuery.queryText(new TextCriteria().matching("\"coffee cake\""));
TextQuery.queryText(new TextCriteria().phrase("coffee cake"));

// 대소문자/악센트 민감도 (MongoDB 3.2+)
TextCriteria criteria = new TextCriteria()
    .matching("coffee")
    .caseSensitive(true)
    .diacriticSensitive(true);
```

---

## Query by Example

특정 도메인 객체를 "이런 모양 비슷한 거"라는 식의 **샘플(probe)** 로 넘겨 비슷한 문서를 찾는 방식이다. 별도의 조건 빌더 없이도 의미가 명확한 쿼리를 만들 수 있어 단순한 룩업에 잘 어울린다.

```java
Person probe = new Person();
probe.lastname = "stark";

Example<Person> example = Example.of(probe);

Query query = new Query(new Criteria().alike(example));
List<Person> result = template.find(query, Person.class);
```

기본 모드는 **타입 정보까지 포함된 strict 모드**이다. 즉, 매핑이 `_class : { $in : [com.acme.Person] }` 같은 타입 힌트를 함께 추가한다. 같은 컬렉션에 여러 도메인 타입을 섞어서 저장하거나 타입 힌트를 끄고 싶다면 `UntypedExampleMatcher`를 쓴다.

```java
class JustAnArbitraryClassWithMatchingFieldName {
    @Field("lastname") String value;
}

JustAnArbitraryClassWithMatchingFieldName probe = new JustAnArbitraryClassWithMatchingFieldName();
probe.value = "stark";

Example<?> example = Example.of(probe, UntypedExampleMatcher.matching());

Query query = new Query(new Criteria().alike(example));
List<Person> result = template.find(query, Person.class);
```

### 문자열 매칭 동작 살펴보기

문자열 필드는 `StringMatcher`로 매칭 방식을 바꿀 수 있다. 어떤 모드를 어떤 대소문자 옵션과 조합하느냐에 따라 만들어지는 쿼리가 다음처럼 달라진다.

| StringMatcher | 대소문자 | 만들어지는 쿼리                                                    |
|---------------|------|-------------------------------------------------------------|
| `DEFAULT`     | 민감   | `{"firstname" : firstname}`                                 |
| `DEFAULT`     | 둔감   | `{"firstname" : { $regex: firstname, $options: 'i'}}`       |
| `EXACT`       | 민감   | `{"firstname" : { $regex: /^firstname$/}}`                  |
| `EXACT`       | 둔감   | `{"firstname" : { $regex: /^firstname$/, $options: 'i'}}`   |
| `STARTING`    | 민감   | `{"firstname" : { $regex: /^firstname/}}`                   |
| `STARTING`    | 둔감   | `{"firstname" : { $regex: /^firstname/, $options: 'i'}}`    |
| `ENDING`      | 민감   | `{"firstname" : { $regex: /firstname$/}}`                   |
| `ENDING`      | 둔감   | `{"firstname" : { $regex: /firstname$/, $options: 'i'}}`    |
| `CONTAINING`  | 민감   | `{"firstname" : { $regex: /.*firstname.*/}}`                |
| `CONTAINING`  | 둔감   | `{"firstname" : { $regex: /.*firstname.*/, $options: 'i'}}` |
| `REGEX`       | 민감   | `{"firstname" : { $regex: /firstname/}}`                    |
| `REGEX`       | 둔감   | `{"firstname" : { $regex: /firstname/, $options: 'i'}}`     |

> 샘플에 `null` 필드가 섞여 있고 그 값을 매칭에 포함하도록 `ExampleMatcher`를 구성하면, 점 표기 대신 임베디드 도큐먼트 매칭이 사용된다. 이 경우 모든 필드 값과 그 순서까지 일치해야 하므로 의도한 결과가 나오지 않을 수 있다. 또, `@TypeAlias`를 쓰는 환경에서는 `MappingContext`를 미리 초기화해 두지 않으면 별칭을 인식하지 못할 수 있어 `initialEntitySet`을 함께 설정한다.

---

## JSON 스키마로 조회

스키마 정의를 만족하는 도큐먼트만 가져오고 싶다면 `MongoJsonSchema`와 `Criteria.matchingDocumentStructure(...)`를 함께 쓴다.

```java
MongoJsonSchema schema = MongoJsonSchema.builder()
    .required("firstname", "lastname")
    .build();

template.find(query(matchingDocumentStructure(schema)), Person.class);
```

> 스키마 작성 자체에 대해서는 [JSON Schema 매핑 문서](https://docs.spring.io/spring-data/mongodb/reference/mongodb/mapping/mapping-schema.html)를 참고하면 된다.
