# Saving, Updating, and Removing Documents

`MongoTemplate`은 도메인 객체를 그대로 받아서 BSON 문서로 저장하고, 그 반대 방향의 매핑까지 한 번에 처리한다. 이 문서는 "이 객체를 컬렉션에 어떻게 집어넣고, 어떻게 바꾸고, 어떻게 지우느냐"에 해당하는 메서드들을 모은 것이다. 조회는 [Querying Documents](./querying-documents.md) 쪽을 참고하면 된다.

> 원문: <https://docs.spring.io/spring-data/mongodb/reference/mongodb/template-crud-operations.html>

이 페이지에서 다루는 주제는 다음과 같다.

| 주제                                                          | 한 줄 요약                              |
|-------------------------------------------------------------|-------------------------------------|
| [도메인 클래스 준비](#도메인-클래스-준비)                                   | 예제에서 계속 등장할 `Person` 클래스의 골격.       |
| [`_id` 필드 매핑](#_id-필드-매핑)                                   | 어떤 자바 필드가 `_id`로 가는가, 그리고 타입 변환 규칙. |
| [Insert와 Save](#insert와-save)                               | "삽입만" 할지, "있으면 덮어쓸지"를 가르는 두 동사.     |
| [일괄 삽입과 Bulk 연산](#일괄-삽입과-bulk-연산)                           | 여러 건을 한 번에 보낼 때 쓸 수 있는 두 가지 모양.     |
| [Update](#update)                                           | `Update` 빌더로 부분 수정 표현하기.            |
| [Aggregation Pipeline Update](#aggregation-pipeline-update) | 다른 필드 값으로부터 새 값을 계산해서 갱신하기.         |
| [Upsert](#upsert)                                           | 있으면 수정, 없으면 삽입.                     |
| [Replace](#replace)                                         | 문서 한 건을 통째로 바꿔 끼우기.                 |
| [findAndModify](#findandmodify)                             | "수정 후 반환"을 한 번의 원자적 연산으로.           |
| [findAndReplace](#findandreplace)                           | "교체 후 반환"을 fluent API로 표현.          |
| [Remove](#remove)                                           | 단건/조건/배치 삭제의 다섯 가지 변형.              |
| [낙관적 락 `@Version`](#낙관적-락-version)                          | 동시 수정 충돌을 버전 번호로 감지하기.              |

> 모든 메서드는 동기(`MongoTemplate`)와 반응형(`ReactiveMongoTemplate`) 양쪽에 같은 이름·같은 인자 모양으로 존재한다. 차이는 반환 타입 정도이며(`T` vs `Mono<T>`, `List<T>` vs `Flux<T>`), 본문에서는 imperative 예시를 기본으로 두고 필요한 곳에서만 reactive 예시를 함께 보여준다.

---

## 도메인 클래스 준비

이후의 모든 예제는 다음 `Person` 클래스를 기준으로 한다. 별도의 어노테이션이 없다는 점에 주목하자. `id`라는 이름의 필드만으로도 `_id` 매핑이 자동으로 잡힌다.

```java
public class Person {

    private String id;
    private String name;
    private int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public String getId() {
        return id;
    }

    public String getName() {
        return name;
    }

    public int getAge() {
        return age;
    }

    @Override
    public String toString() {
        return "Person [id=" + id + ", name=" + name + ", age=" + age + "]";
    }
}
```

저장될 컬렉션 이름은 기본적으로 **클래스 이름의 첫 글자를 소문자로 바꾼 형태**이다. 즉 `com.test.Person`은 `person` 컬렉션에 저장된다. 다른 이름을 쓰고 싶다면 `@Document(collection = "...")`로 클래스에 박아두거나, 메서드 호출 시 마지막 인자로 컬렉션 이름을 직접 넘기면 된다.

---

## `_id` 필드 매핑

MongoDB는 모든 도큐먼트에 `_id` 필드를 요구한다. Spring Data는 어떤 자바 필드를 `_id`로 매핑할지 다음 순서로 결정한다.

1. `@Id`(`org.springframework.data.annotation.Id`)가 붙은 필드/프로퍼티가 있으면 그것을 `_id`로 매핑한다.
2. 어노테이션이 없더라도 이름이 `id`인 필드/프로퍼티가 있으면 그것을 `_id`로 매핑한다.

둘 중 어느 것도 해당하지 않으면 드라이버가 암묵적으로 `_id`를 만들어 박아두지만, 그 값은 자바 필드 어디에도 매핑되지 않는다.

### 타입 변환 규칙

`_id`로 매핑된 필드의 자바 타입에 따라 저장 시 변환이 일어난다.

| 자바 타입        | 동작                                                                                |
|--------------|-----------------------------------------------------------------------------------|
| `String`     | 가능하면 `ObjectId`로 변환해 저장한다. 변환할 수 없는 문자열이면 그대로 문자열로 저장된다.                          |
| `Date`       | `ObjectId`로 변환해 저장한다.                                                             |
| `BigInteger` | `ObjectId`로 변환해 저장한다.                                                             |
| `ObjectId`   | 그대로 저장된다.                                                                         |

`id`가 비어 있는 채로 insert/save 하면 드라이버가 `ObjectId`를 자동 생성해 채워준다. 단, 자동 생성을 받으려면 자바 필드 타입이 `String`, `ObjectId`, `BigInteger` 중 하나여야 한다.

### `@MongoId`로 더 세밀하게 제어하기

문자열 `id`가 항상 `ObjectId`로 변환되는 것이 부담스러울 때, 또는 반대로 `_id`를 `ObjectId`로 명확히 고정하고 싶을 때는 `@MongoId`를 쓴다.

```java
public class PlainStringId {
    @MongoId String id;                    // (1)
}

public class PlainObjectId {
    @MongoId ObjectId id;                  // (2)
}

public class StringToObjectId {
    @MongoId(FieldType.OBJECT_ID) String id; // (3)
}
```

1. `String` 그대로 저장된다. 변환을 시도하지 않는다.
2. `ObjectId`로 저장된다.
3. 문자열이 유효한 `ObjectId` hex라면 `ObjectId`로, 아니면 `String`으로 저장된다. `@Id`와 동일한 동작이다.

> 한 마디로, `@Id`만 쓰면 "가능하면 변환"이고, `@MongoId`는 변환 정책을 명시적으로 골라잡는 도구라고 보면 된다.

---

## Insert와 Save

부분 수정이 아니라 도큐먼트 한 건을 컬렉션에 통째로 넣고 싶을 때 쓰는 두 메서드가 `insert`와 `save`이다. 이름은 비슷하지만 동작이 다르다.

| 메서드      | 같은 `_id`가 이미 있을 때                        |
|----------|------------------------------------------|
| `insert` | **에러를 낸다.** "넣기"만을 의도하는 호출이다.            |
| `save`   | **덮어쓴다.** 없으면 새로 삽입, 있으면 그 자리를 통째로 갱신한다. |

즉 `save`는 "insert-or-overwrite"이고, `insert`는 "insert-only"라고 외워두면 된다.

### 시그니처

```java
void save(Object objectToSave);
void save(Object objectToSave, String collectionName);

void insert(Object objectToSave);
void insert(Object objectToSave, String collectionName);

<T> Collection<T> insertAll(Collection<? extends T> objectsToSave);
<T> Collection<T> insertAll(Collection<? extends T> objectsToSave, String collectionName);
```

### 사용 예시

**Imperative**

```java
import static org.springframework.data.mongodb.core.query.Criteria.where;
import static org.springframework.data.mongodb.core.query.Query.query;

template.insert(new Person("Bob", 33));

Person person = template.query(Person.class)
    .matching(query(where("age").is(33)))
    .oneValue();
```

**Reactive**

```java
Mono<Person> person = mongoTemplate.insert(new Person("Bob", 33))
    .then(mongoTemplate.query(Person.class)
        .matching(query(where("age").is(33)))
        .one());
```

`Bob`을 삽입하고, 그 직후 `age`가 33인 도큐먼트를 한 건 가져온다. 자바 객체에는 `id`가 비어 있었지만, 삽입 결과로 자동 생성된 `ObjectId`가 들어와 채워진다.

> `@Version`이 붙은 프로퍼티가 비어 있는 상태로 `insert`를 부르면 자동으로 초기화된다. `int`, `long` 같은 원시 타입은 `1`로, `Integer`, `Long` 같은 래퍼 타입은 `0`으로 들어간다.

---

## 일괄 삽입과 Bulk 연산

여러 건을 한 번에 보내고 싶을 때 두 가지 길이 있다. **batch insert**는 같은 도큐먼트들을 한 번의 명령으로 묶어 보내는 단순 형태이고, **bulk operation**은 "insert/update/remove를 섞어서 한 번에" 보내는 좀 더 일반적인 형태다.

### Batch Insert

**Imperative**

```java
Collection<Person> inserted = template.insert(List.of(...), Person.class);
```

**Reactive**

```java
Flux<Person> inserted = template.insert(List.of(...), Person.class);
```

### Bulk Operations

```java
// Imperative
BulkWriteResult result = template.bulkOps(BulkMode.ORDERED, Person.class)
    .insert(List.of(...))
    .execute();

// Reactive
Mono<BulkWriteResult> result = template.bulkOps(BulkMode.ORDERED, Person.class)
    .insert(List.of(...))
    .execute();
```

| `BulkMode`    | 의미                                                |
|---------------|---------------------------------------------------|
| `ORDERED`     | 순서대로 실행하다가 하나가 실패하면 멈춘다. 이전까지의 작업은 적용된 상태로 남는다.   |
| `UNORDERED`   | 가능한 만큼 병렬·순서 무관하게 실행한다. 일부가 실패해도 나머지는 시도한다.       |

`bulkOps(...)`로 만든 빌더에는 `insert(...)`, `updateOne(...)`, `updateMulti(...)`, `upsert(...)`, `remove(...)` 같은 메서드를 자유롭게 체이닝하고 마지막에 `execute()`로 보낸다.

> 서버 입장에서 batch와 bulk의 성능은 동일하다. 다만 **bulk는 라이프사이클 이벤트(`BeforeSave`, `AfterSave` 등)를 발행하지 않는다**. 감사(audit)나 변환 콜백을 의지하고 있다면 batch insert를 쓰는 편이 안전하다.

---

## Update

이미 들어가 있는 도큐먼트의 일부 필드만 바꾸고 싶을 때 쓴다. fluent API의 진입점은 `update(...)`이고, 조건은 `matching(...)`, 변경 내용은 `apply(new Update()...)`로, 종료는 `first()` / `all()`로 닫는다.

```java
// Imperative
UpdateResult result = template.update(Account.class)
    .matching(where("accounts.accountType").is(Type.SAVINGS))
    .apply(new Update().inc("accounts.$.balance", 50.00))
    .all();

// Reactive
Mono<UpdateResult> result = template.update(Account.class)
    .matching(where("accounts.accountType").is(Type.SAVINGS))
    .apply(new Update().inc("accounts.$.balance", 50.00))
    .all();
```

종료 메서드는 두 가지다.

| 종료 메서드    | 의미                        |
|-----------|---------------------------|
| `first()` | 조건에 맞는 **첫 한 건**만 갱신한다.   |
| `all()`   | 조건에 맞는 **모든 도큐먼트**를 갱신한다. |

전통적인 시그니처를 직접 쓰고 싶다면 `updateFirst(Query, Update, Class<?>)`, `updateMulti(Query, Update, Class<?>)`도 그대로 사용할 수 있다.

### `Update` 빌더의 메서드들

`Update`는 MongoDB의 update 연산자에 1:1로 대응하는 메서드를 제공한다.

| 메서드                                       | 대응 연산자         | 설명                                                    |
|-------------------------------------------|----------------|-------------------------------------------------------|
| `set(String key, Object value)`           | `$set`         | 필드 값을 지정한 값으로 설정한다.                                   |
| `setOnInsert(String key, Object value)`   | `$setOnInsert` | upsert가 insert로 동작했을 때만 적용된다.                         |
| `unset(String key)`                       | `$unset`       | 필드를 제거한다.                                             |
| `inc(String key, Number inc)`             | `$inc`         | 숫자 필드에 값을 더한다(음수면 뺀다).                                |
| `multiply(String key, Number multiplier)` | `$mul`         | 숫자 필드에 값을 곱한다.                                        |
| `max(String key, Object max)`             | `$max`         | 현재 값보다 큰 값으로만 갱신한다.                                   |
| `min(String key, Object min)`             | `$min`         | 현재 값보다 작은 값으로만 갱신한다.                                  |
| `currentDate(String key)`                 | `$currentDate` | 현재 일시(`Date`)로 설정한다.                                  |
| `currentTimestamp(String key)`            | `$currentDate` | 현재 시각을 BSON `Timestamp`로 설정한다.                        |
| `push(String key, Object value)`          | `$push`        | 배열 끝에 원소를 추가한다.                                       |
| `pushAll(String key, Object[] values)`    | `$pushAll`     | 배열 끝에 여러 원소를 추가한다(deprecated, `push().each(...)` 권장). |
| `addToSet(String key, Object value)`      | `$addToSet`    | 배열에 중복 없이 원소를 추가한다.                                   |
| `pop(String key, Update.Position pos)`    | `$pop`         | 배열의 처음/끝 원소를 제거한다.                                    |
| `pull(String key, Object value)`          | `$pull`        | 조건에 맞는 원소를 배열에서 제거한다.                                 |
| `pullAll(String key, Object[] values)`    | `$pullAll`     | 주어진 값들과 일치하는 원소를 모두 제거한다.                             |
| `rename(String oldName, String newName)`  | `$rename`      | 필드 이름을 바꾼다.                                           |

### `push`와 `addToSet`의 중첩 옵션

배열 작업은 단순히 "한 개 추가"보다 더 자주 쓰이는 모양들이 있다. 빌더가 그 형태들을 그대로 표현해 준다.

```java
// { $push : { "category" : { "$each" : [ "spring" , "data" ] } } }
new Update().push("category").each("spring", "data");

// { $push : { "key" : { "$position" : 0 , "$each" : [ "Arya" , "Arry" , "Weasel" ] } } }
new Update().push("key").atPosition(Position.FIRST).each(Arrays.asList("Arya", "Arry", "Weasel"));

// { $push : { "key" : { "$slice" : 5 , "$each" : [ "Arya" , "Arry" , "Weasel" ] } } }
new Update().push("key").slice(5).each(Arrays.asList("Arya", "Arry", "Weasel"));

// { $addToSet : { "values" : { "$each" : [ "spring" , "data" , "mongodb" ] } } }
new Update().addToSet("values").each("spring", "data", "mongodb");
```

`each(...)`는 `$each`로, `atPosition(...)`은 `$position`으로, `slice(...)`는 `$slice`로 그대로 매핑된다.

---

## Aggregation Pipeline Update

MongoDB 4.2부터는 update 자리에 집계(aggregation) 파이프라인을 줄 수 있다. **다른 필드 값을 참조해 새 값을 계산하는** 갱신이 필요할 때 강력하다. Spring Data에서는 `AggregationUpdate`로 만든다.

```java
AggregationUpdate update = Aggregation.newUpdate()
    .set("average").toValue(ArithmeticOperators.valueOf("tests").avg())     // (1)
    .set("grade").toValue(ConditionalOperators.switchCases(                 // (2)
        when(valueOf("average").greaterThanEqualToValue(90)).then("A"),
        when(valueOf("average").greaterThanEqualToValue(80)).then("B"),
        when(valueOf("average").greaterThanEqualToValue(70)).then("C"),
        when(valueOf("average").greaterThanEqualToValue(60)).then("D"))
        .defaultTo("F")
    );

template.update(Student.class)                                              // (3)
    .apply(update)
    .all();                                                                 // (4)
```

1. `tests` 배열의 평균을 구해 `average` 필드에 박는다.
2. 방금 계산한 `average`를 다시 참조해 등급(`grade`) 문자열을 결정한다.
3. fluent API의 update 진입점.
4. 종료 메서드 호출 시점에 실제 명령이 떠난다.

지원되는 단계는 다음 셋이다.

| 빌더 호출                                     | 생성되는 단계                  |
|-------------------------------------------|--------------------------|
| `AggregationUpdate.set(...).toValue(...)` | `$set : { ... }`         |
| `AggregationUpdate.unset(...)`            | `$unset : [ ... ]`       |
| `AggregationUpdate.replaceWith(...)`      | `$replaceWith : { ... }` |

---

## Upsert

조건에 맞는 도큐먼트가 있으면 갱신하고, 없으면 새로 삽입하는 동작이다. fluent API의 종료 메서드를 `upsert()`로 닫으면 된다.

```java
// Imperative
UpdateResult result = template.update(Person.class)
    .matching(query(where("ssn").is(1111)
        .and("firstName").is("Joe")
        .and("Fraizer").is("Update")))
    .apply(update("address", addr))
    .upsert();

// Reactive
Mono<UpdateResult> result = template.update(Person.class)
    .matching(query(where("ssn").is(1111)
        .and("firstName").is("Joe")
        .and("Fraizer").is("Update")))
    .apply(update("address", addr))
    .upsert();
```

> `upsert`는 **정렬을 지원하지 않는다.** 정렬을 함께 쓰고 싶다면 `findAndModify`(아래)에서 `Sort`를 거는 편으로 우회한다.

`@Version` 프로퍼티가 `Update`에 포함되어 있지 않더라도 자동으로 초기화된다(원시 타입이면 `1`, 래퍼 타입이면 `0`).

---

## Replace

`Update`로는 표현이 거추장스러운, **문서 한 건을 통째로 바꿔 끼우는** 작업이다. 부분 수정이 아니라 "기존 문서를 새 문서로 대체"하는 의미다.

### 단건 교체

```java
Person tom = template.insert(new Person("Motte", 21));            // (1)
Query query = Query.query(Criteria.where("firstName").is(tom.getFirstName())); // (2)
tom.setFirstname("Tom");                                          // (3)
template.replace(query, tom, ReplaceOptions.none());              // (4)
```

1. 도큐먼트를 하나 만들어 둔다.
2. 조회 조건을 만든다.
3. 자바 객체 쪽에서 값을 고친다.
4. 조건에 맞는 한 건을 통째로 `tom`으로 갈아 끼운다.

### Upsert와 함께 쓰기

```java
Person tom = new Person("id-123", "Tom", 21);                     // (1)
Query query = Query.query(Criteria.where("firstName").is(tom.getFirstName()));
template.replace(query, tom, ReplaceOptions.replaceOptions().upsert()); // (2)
```

1. **upsert로 동작하려면 `_id`가 반드시 들어 있어야 한다.**
2. 조건에 맞는 도큐먼트가 없으면 이 객체를 그대로 삽입한다.

> 교체 대상 도큐먼트의 `_id`는 기존 값과 같거나, 아예 없어야 한다. **replace로는 `_id`를 바꿀 수 없다.** upsert에서 쿼리에도 교체 객체에도 `_id`가 없으면 MongoDB가 새 `ObjectId`를 만들어 넣는데, 도메인 타입이 다른 id 타입(예: `Long`)을 기대하고 있으면 매핑 단계에서 문제가 생기니 주의해야 한다.

---

## findAndModify

"수정 후, 그 결과를 받아본다"를 한 번의 원자적 연산으로 처리한다. 동시 갱신 환경에서 안전하게 "내가 갱신한 그 값"을 받아오고 싶을 때 쓴다.

### 시그니처

```java
<T> T findAndModify(Query query, Update update, Class<T> entityClass);
<T> T findAndModify(Query query, Update update, Class<T> entityClass, String collectionName);
<T> T findAndModify(Query query, Update update, FindAndModifyOptions options, Class<T> entityClass);
<T> T findAndModify(Query query, Update update, FindAndModifyOptions options, Class<T> entityClass, String collectionName);
```

### Fluent API 예시

```java
template.insert(new Person("Tom", 21));
template.insert(new Person("Dick", 22));
template.insert(new Person("Harry", 23));

Query query = new Query(Criteria.where("firstName").is("Harry"));
Update update = new Update().inc("age", 1);

Person oldValue = template.update(Person.class)
    .matching(query)
    .apply(update)
    .findAndModifyValue();              // oldValue.age == 23

Person newValue = template.query(Person.class)
    .matching(query)
    .findOneValue();                    // newValue.age == 24

Person newestValue = template.update(Person.class)
    .matching(query)
    .apply(update)
    .withOptions(FindAndModifyOptions.options().returnNew(true))
    .findAndModifyValue();              // newestValue.age == 25
```

기본 동작은 **갱신 직전의 도큐먼트(이전 값)** 를 돌려준다. `returnNew(true)`로 옵션을 켜면 갱신 직후의 값을 받는다.

### 옵션

| 옵션                | 의미                                                                      |
|-------------------|-------------------------------------------------------------------------|
| `returnNew(true)` | 갱신 후 값을 반환한다(기본은 갱신 전 값).                                               |
| `upsert(true)`    | 일치하는 문서가 없으면 삽입한다. `upsert`+`returnNew`로 "있으면 갱신, 없으면 삽입한 결과 받아오기"를 표현. |
| `remove()`        | 갱신 대신 제거하고, 제거된 도큐먼트를 반환한다.                                             |

```java
Person upserted = template.update(Person.class)
    .matching(new Query(Criteria.where("firstName").is("Mary")))
    .apply(update)
    .withOptions(FindAndModifyOptions.options().upsert(true).returnNew(true))
    .findAndModifyValue();
```

---

## findAndReplace

`findAndModify`와 형이 닮았지만, **부분 수정이 아니라 전체 교체** 라는 점이 다르다. fluent API에서 `replaceWith(...)`를 함께 쓰는 모양이 진입점이다.

```java
Optional<User> result = template.update(Person.class)               // (1)
    .matching(query(where("firstame").is("Tom")))                   // (2)
    .replaceWith(new Person("Dick"))
    .withOptions(FindAndReplaceOptions.options().upsert())          // (3)
    .as(User.class)                                                 // (4)
    .findAndReplace();                                              // (5)
```

1. fluent update API의 진입점. 이 도메인 타입을 기준으로 쿼리/매핑이 일어난다.
2. 실제 매칭 쿼리. `sort`, `fields`, `collation`도 함께 걸 수 있다.
3. `upsert` 같은 추가 옵션을 줄 수 있다.
4. **결과를 다른 타입으로 받고 싶을 때** 사용하는 프로젝션. 생략하면 진입점에서 지정한 도메인 타입을 그대로 쓴다.
5. 실행 트리거. `Optional` 대신 nullable 값을 직접 받고 싶다면 `findAndReplaceValue()`를 쓴다.

---

## Remove

도큐먼트 삭제는 다섯 가지 모양이 있다.

```java
template.remove(tywin, "GOT");                                              // (1)

template.remove(query(where("lastname").is("lannister")), "GOT");           // (2)

template.remove(new Query().limit(3), "GOT");                               // (3)

template.findAllAndRemove(query(where("lastname").is("lannister")), "GOT"); // (4)

template.findAllAndRemove(new Query().limit(3), "GOT");                     // (5)
```

1. 도메인 객체의 `_id`로 한 건을 지운다.
2. 조건에 맞는 도큐먼트를 모두 지운다.
3. 조건/정렬/limit/skip을 적용해 식별된 도큐먼트들을 **한 번의 batch 명령**으로 지운다.
4. (2)와 같은 의미지만 도큐먼트들을 **하나씩** 지우면서 그 객체들을 함께 받아온다.
5. (3)과 같은 의미지만 batch가 아니라 한 건씩 지운다.

반환 타입은 형태마다 다르다.

| 메서드                                                | 반환 타입          |
|----------------------------------------------------|----------------|
| `remove(Object entity)` / `remove(Object, String)` | `void`         |
| `remove(Query, Class<?>)`                          | `DeleteResult` |
| `findAllAndRemove(Query, Class<T>)`                | `List<T>`      |

> "지우면서 데이터를 받아 두고 싶다"면 `findAllAndRemove`, "삭제 카운트만 알면 된다"면 `remove(Query, ...)`를 고른다. 단순히 객체 한 건만 정리한다면 `remove(entity)`가 가장 짧다.

---

## 낙관적 락 `@Version`

여러 작업자가 같은 도큐먼트를 동시에 수정할 때, "마지막에 저장한 사람이 이긴다"가 되지 않도록 막는 장치가 낙관적 락이다. 도메인에 `@Version` 필드를 두면, Spring Data가 그 필드를 매 갱신마다 1씩 올리고, 갱신 시 "내가 알고 있는 버전"이 서버의 현재 버전과 다르면 예외를 던진다.

```java
@Document
class Person {

    @Id String id;
    String firstname;
    String lastname;

    @Version Long version;
}
```

### 실제 동작 흐름

```java
Person daenerys = template.insert(new Person("Daenerys"));                            // (1)

Person tmp = template.findOne(query(where("id").is(daenerys.getId())), Person.class); // (2)

daenerys.setLastname("Targaryen");
template.save(daenerys);                                                              // (3)

template.save(tmp); // throws OptimisticLockingFailureException                       // (4)
```

1. 최초 삽입. `version`이 `0`으로 들어간다.
2. 방금 넣은 도큐먼트를 다시 읽어둔다. `tmp`의 `version`은 여전히 `0`.
3. `daenerys` 객체를 갱신해 저장한다. 서버상의 `version`은 `1`로 올라간다.
4. 하지만 `tmp`는 자기 손에 든 `version`이 `0`이라고 알고 있다. 서버는 `1`이므로 일치하지 않는다 → `OptimisticLockingFailureException`이 던져진다.

### 알아두면 좋은 동작 규칙

- 모든 CRUD 메서드가 `@Version`을 고려하지는 않는다. 자세한 매트릭스는 `MongoOperations` Javadoc을 참고.
- `save`, `updateFirst`, `updateMulti`, `findAndModify`는 자동으로 버전을 증가시킨다.
- `Update`에 `version`을 명시하지 않아도 자동 증가된다.
- 낙관적 락은 **서버의 write acknowledgement** 가 있어야 동작한다. `WriteConcern.UNACKNOWLEDGED`를 쓰면 충돌이 조용히 무시될 수 있다.
- 2.2 버전부터 **엔티티 단위의 `remove(entity)`도 버전을 검사**한다. 버전 검사 없이 지우려면 `remove(Query, ...)` 형태를 사용한다.
- 리포지토리의 `CrudRepository.delete(Object)`도 같은 검사를 거친다. 우회하려면 `deleteById(ID)`를 쓴다.

> 낙관적 락이 던지는 예외는 `OptimisticLockingFailureException` 한 종류이며, 의미는 "당신이 들고 있는 버전이 더 이상 최신이 아니다"이다. 보통은 "다시 읽어와서 사용자에게 충돌을 알리고 재시도"하는 식으로 받아 처리한다.
