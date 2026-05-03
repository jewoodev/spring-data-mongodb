> **원문 저작권:** Copyright © 2008-2025 VMware, Inc. (Broadcom Inc.). All rights reserved.
>
> 본 문서의 사본은 개인적인 용도 및 타인에게 배포하는 용도로 제작할 수 있습니다.
> 단, 사본에 대해 어떠한 수수료도 부과해서는 안 되며, 인쇄물이든 전자적 형태이든
> 배포하는 모든 사본에는 본 저작권 고지가 포함되어야 합니다.
>
> **원문:** <https://docs.spring.io/spring-data/mongodb/reference/mongodb/mapping/mapping.html>
>
> 본 문서는 학습 목적의 비공식 한국어 번역이며, 정확한 내용은 원문을 참조하시기 바랍니다.

# Spring Data MongoDB - Object Mapping

이 문서는 [Spring Data MongoDB의 Object Mapping 공식 레퍼런스](https://docs.spring.io/spring-data/mongodb/reference/mongodb/mapping/mapping.html)를 한국어로 풀어서 정리한 학습 노트다. Spring Data MongoDB가 도메인 객체와 MongoDB 도큐먼트 사이를 어떻게 오가게 하는지, 그 매커니즘과 설정 방법을 다룬다.

핵심 컴포넌트는 `MappingMongoConverter`다. 이 컨버터는 도메인 객체를 BSON 도큐먼트로(또는 그 반대로) 변환하기 위한 메타데이터 모델을 들고 있다. 메타데이터는 보통 어노테이션으로 채우지만, 어노테이션이 없어도 몇 가지 컨벤션만으로 매핑이 가능하도록 설계되어 있다.

---

## 1. Object Mapping 기초

이 챕터는 Spring Data 공통의 객체 매핑 원리(인스턴스 생성, 필드/프로퍼티 접근, 가변/불변 처리)를 다룬다. JPA처럼 별도 매핑 레이어를 두지 않는 Spring Data 모듈들에 공통적으로 적용되는 내용이다. 인덱스나 필드명 커스터마이징처럼 저장소별로 다른 부분은 각 모듈의 별도 섹션에서 다룬다.

Spring Data의 Object Mapping이 책임지는 핵심 두 가지는 다음과 같다.

1. **인스턴스 생성** - 클래스에 노출된 생성자 중 하나를 골라 객체를 만든다.
2. **프로퍼티 채우기** - 노출된 프로퍼티들에 값을 입힌다.

### 1.1 객체 생성 (Object Creation)

Spring Data는 영속 엔티티의 생성자를 자동으로 탐지한다. 어떤 생성자를 쓸지 결정하는 알고리즘은 다음과 같다.

1. `@PersistenceCreator`가 붙은 단 하나의 정적 팩토리 메서드가 있다면 그것을 쓴다.
2. 생성자가 단 하나만 있다면 그것을 쓴다.
3. 생성자가 여러 개인데 그 중 하나에만 `@PersistenceCreator`가 붙어 있다면 그것을 쓴다.
4. 타입이 자바 `Record`이면 canonical constructor를 쓴다.
5. no-argument 생성자가 있다면 그것을 쓰고 나머지는 무시한다.

값을 매핑할 때, 생성자나 팩토리 메서드의 **파라미터 이름이 엔티티의 프로퍼티 이름과 일치한다**고 가정한다. 즉, 매핑 단계에서 적용되는 모든 커스터마이징(필드명 변경 등)이 반영된 후의 프로퍼티 이름과 매칭된다. 이게 동작하려면 컴파일된 클래스 파일에 파라미터 이름 정보가 살아 있거나, 생성자에 `@ConstructorProperties` 어노테이션이 붙어 있어야 한다.

값 매핑은 Spring Framework의 `@Value` 어노테이션과 저장소별 SpEL 표현식으로 커스터마이징할 수 있다.

#### 객체 생성 내부 구조

리플렉션 오버헤드를 피하기 위해, Spring Data는 런타임에 팩토리 클래스를 생성해서 도메인 클래스의 생성자를 직접 호출한다. 예를 들어 다음과 같은 클래스가 있을 때

```java
class Person {
  Person(String firstname, String lastname) { … }
}
```

런타임에 다음과 의미적으로 동등한 팩토리 클래스가 만들어진다.

```java
class PersonObjectInstantiator implements ObjectInstantiator {

  Object newInstance(Object... args) {
    return new Person((String) args[0], (String) args[1]);
  }
}
```

이 방식은 리플렉션보다 약 10% 더 빠르다. 다만 이 최적화의 혜택을 받으려면 도메인 클래스가 다음 조건을 만족해야 한다.

- private 클래스가 아니어야 한다
- non-static inner class가 아니어야 한다
- CGLib 프록시 클래스가 아니어야 한다
- Spring Data가 사용할 생성자가 private이 아니어야 한다

조건 중 하나라도 어긋나면 Spring Data는 리플렉션 기반 인스턴스화로 fallback한다.

### 1.2 프로퍼티 채우기 (Property Population)

인스턴스가 만들어진 다음, Spring Data는 남은 영속 프로퍼티들을 채워 나간다. 이때 식별자(identifier) 프로퍼티가 가장 먼저 채워지는데, 이는 순환 참조를 풀어내기 위함이다(생성자에서 이미 식별자가 들어왔다면 이 단계는 생략된다). 그 후 생성자에서 채워지지 않은 비-transient 프로퍼티들이 다음 알고리즘에 따라 채워진다.

1. 프로퍼티가 불변이지만 `with…` 메서드를 노출한다면, 그 메서드를 호출해 새 인스턴스를 만든다.
2. 프로퍼티 접근(getter/setter)이 정의되어 있다면 setter를 호출한다.
3. 프로퍼티가 가변이라면 필드를 직접 set한다.
4. 프로퍼티가 불변이라면 영속용 생성자를 호출해 새 복사본을 만든다.
5. 그 외에 경우에는 기본적으로 필드를 직접 set한다.

#### 프로퍼티 채우기 내부 구조

객체 생성과 마찬가지로, 프로퍼티 접근에도 런타임 생성 접근자 클래스를 사용한다. 다음과 같은 엔티티가 있을 때

```java
class Person {

  private final Long id;
  private String firstname;
  private @AccessType(Type.PROPERTY) String lastname;

  Person() {
    this.id = null;
  }

  Person(Long id, String firstname, String lastname) {
    // 필드 할당
  }

  Person withId(Long id) {
    return new Person(id, this.firstname, this.lastname);
  }

  void setLastname(String lastname) {
    this.lastname = lastname;
  }
}
```

생성되는 프로퍼티 접근자는 대략 이런 모습이다.

```java
class PersonPropertyAccessor implements PersistentPropertyAccessor {

  private static final MethodHandle firstname;              // 2

  private Person person;                                    // 1

  public void setProperty(PersistentProperty property, Object value) {

    String name = property.getName();

    if ("firstname".equals(name)) {
      firstname.invoke(person, (String) value);             // 2
    } else if ("id".equals(name)) {
      this.person = person.withId((Long) value);            // 3
    } else if ("lastname".equals(name)) {
      this.person.setLastname((String) value);              // 4
    }
  }
}
```

1. PropertyAccessor는 내부 객체의 가변 인스턴스를 들고 있다. 이렇게 해야 불변 프로퍼티의 변경(즉, 새 인스턴스로의 교체)을 추적할 수 있다.
2. 기본 동작은 필드 직접 접근이다. private 필드의 가시성 문제를 풀기 위해 `MethodHandles`를 통해 필드와 상호작용한다.
3. `withId(…)` 메서드는 새 식별자가 부여될 때(예: 데이터스토어 삽입 후 자동 생성된 ID 적용) 새 `Person` 인스턴스를 반환한다. 이후 변경은 모두 이 새 인스턴스에서 일어난다.
4. `@AccessType(Type.PROPERTY)`로 지정된 프로퍼티는 `MethodHandles`를 거치지 않고 메서드를 직접 호출한다.

이 방식은 리플렉션 대비 약 25%의 성능 향상이 있다. 이 최적화를 받으려면 다음 조건을 만족해야 한다.

- 타입이 default 패키지나 `java` 패키지 아래에 있으면 안 된다
- 타입과 그 생성자가 모두 `public`이어야 한다
- 내부 클래스라면 `static`이어야 한다
- 사용 중인 자바 런타임이 originating `ClassLoader`에서 클래스 선언을 허용해야 한다 (Java 9 이상에서 일부 제한 있음)

조건이 안 맞으면 Spring Data는 리플렉션 기반 접근자로 fallback한다.

#### 종합 예시

위 규칙들이 적용된 모범 엔티티는 대략 이런 모습이다.

```java
class Person {

  private final @Id Long id;                                                // 1
  private final String firstname, lastname;                                 // 2
  private final LocalDate birthday;
  private final int age;                                                    // 3

  private String comment;                                                   // 4
  private @AccessType(Type.PROPERTY) String remarks;                        // 5

  static Person of(String firstname, String lastname, LocalDate birthday) { // 6

    return new Person(null, firstname, lastname, birthday,
      Period.between(birthday, LocalDate.now()).getYears());
  }

  Person(Long id, String firstname, String lastname, LocalDate birthday, int age) { // 6

    this.id = id;
    this.firstname = firstname;
    this.lastname = lastname;
    this.birthday = birthday;
    this.age = age;
  }

  Person withId(Long id) {                                                  // 1
    return new Person(id, this.firstname, this.lastname, this.birthday, this.age);
  }

  void setRemarks(String remarks) {                                         // 5
    this.remarks = remarks;
  }
}
```

1. **식별자는 final이지만 생성자에서 `null`로 초기화된다.** 데이터스토어 삽입 후 식별자가 부여되면 `withId(…)`가 새 인스턴스를 반환한다. 원본 `Person`은 그대로 남는다. wither 메서드는 사실 선택 사항인데, all-args persistence constructor가 효과적인 copy constructor 역할을 하기 때문이다.
2. **`firstname`, `lastname`은 평범한 불변 프로퍼티다.** getter로 노출될 수 있다.
3. **`age`는 `birthday`에서 파생된 불변·계산 프로퍼티다.** Spring Data는 이 클래스에 단 하나뿐인 생성자를 사용하므로, 데이터베이스에 저장된 `age` 값이 계산값보다 우선 적용된다. 계산을 늘 새로 하고 싶더라도, 생성자가 `age`를 받지 않으면 프로퍼티 채우기 단계에서 final 필드를 set하려다가 실패하므로, 일단 받아 두고 무시할 여지가 필요하다.
4. **`comment`는 가변이며 필드 직접 접근으로 채워진다.**
5. **`remarks`는 가변이며 setter 호출로 채워진다.**
6. **객체 생성용 팩토리 메서드와 (영속용) 생성자를 함께 노출한다.** 핵심 아이디어는 추가 생성자로 인한 모호함(그래서 `@PersistenceCreator`가 필요해지는 상황)을 피하고, 기본값 세팅은 팩토리 메서드 안에서 처리하는 것이다. Spring Data가 팩토리 메서드를 인스턴스화에 쓰게 하려면 `@PersistenceCreator`를 그쪽에 붙이면 된다.

### 1.3 일반적인 권장 사항

- **불변 객체를 우선 고려하라.** 생성자 호출만으로 인스턴스화가 끝나서 단순하고 빠르다(프로퍼티 채우기보다 최대 30% 빠름). 도메인 객체에 setter가 난립하는 것도 막을 수 있다. 부득이하게 setter가 필요하면 같은 패키지에서만 쓸 수 있도록 package-protected로 두는 게 낫다.
- **all-args 생성자를 제공하라.** 불변으로 만들기 어려운 엔티티라도, 모든 프로퍼티를 받는 생성자를 두면 매핑 단계에서 프로퍼티 채우기를 건너뛸 수 있어 성능이 좋아진다.
- **`@PersistenceCreator`를 피하기 위해 생성자 오버로드 대신 정적 팩토리 메서드를 사용하라.** 자동 생성 ID처럼 일부 인자를 생략한 생성자를 노출하고 싶은 경우, 생성자를 늘리는 대신 팩토리 메서드를 쓰는 것이 정착된 패턴이다.
- **앞서 언급한 instantiator/property accessor 생성 조건을 지켜라.** 그래야 리플렉션이 아닌 최적화된 경로를 탄다.
- **자동 생성되는 식별자라면 final 필드로 두고, all-args persistence constructor의 파라미터로 받거나 `with…` 메서드를 함께 제공하라.**
- **보일러플레이트는 Lombok으로 줄여라.** 영속 작업은 보통 all-args 생성자가 필요한데, 이를 일일이 작성하는 건 지겨운 일이다. Lombok의 `@AllArgsConstructor`가 가장 깔끔한 해결책이다.

### 1.4 프로퍼티 오버라이딩 (Overriding Properties)

자바는 서브클래스가 상위 클래스에 이미 존재하는 이름의 프로퍼티를 다시 선언할 수 있게 해 준다. 다음 예시를 보자.

```java
public class SuperType {

   private CharSequence field;

   public SuperType(CharSequence field) {
      this.field = field;
   }

   public CharSequence getField() {
      return this.field;
   }

   public void setField(CharSequence field) {
      this.field = field;
   }
}

public class SubType extends SuperType {

   private String field;

   public SubType(String field) {
      super(field);
      this.field = field;
   }

   @Override
   public String getField() {
      return this.field;
   }

   public void setField(String field) {
      this.field = field;

      // optional
      super.setField(field);
   }
}
```

두 클래스 모두 호환 가능한 타입으로 `field`를 정의하고 있지만, `SubType`이 `SuperType.field`를 가린다(shadow). 설계에 따라 `SuperType.field`를 채우는 유일한 기본 경로는 생성자이거나, setter 안에서 `super.setField(...)`를 함께 호출하는 것이다. 어느 쪽이든 같은 이름으로 두 개의 다른 값이 존재할 수 있다는 모순이 생긴다.

Spring Data는 서브타입이 슈퍼타입에 할당 가능하지 않은 경우 슈퍼타입 프로퍼티를 건너뛴다. 즉, 오버라이드로 등록되려면 오버라이드된 프로퍼티의 타입이 원본 프로퍼티 타입에 할당 가능해야 하고, 그렇지 않으면 슈퍼타입 프로퍼티는 transient로 간주된다. **가능하면 서로 다른 이름을 쓰는 것이 가장 안전하다.**

오버라이드된 프로퍼티를 다룰 때 고려해야 할 점은 다음과 같다.

1. **어떤 프로퍼티를 영속화할 것인가?** 기본은 선언된 모든 프로퍼티가 대상이다. `@Transient`로 제외할 수 있다.
2. **데이터스토어에서 어떻게 표현할 것인가?** 같은 필드/컬럼 이름을 다른 값에 그대로 쓰면 데이터가 깨질 수 있으니, 적어도 한쪽에는 명시적인 필드/컬럼 이름을 어노테이션으로 지정해야 한다.
3. `@AccessType(PROPERTY)`는 사용할 수 없다. setter 구현에 추가 가정 없이는 슈퍼타입 프로퍼티를 일반적으로 set할 수 없기 때문이다.

### 1.5 Kotlin 지원

Kotlin 클래스도 인스턴스화 대상이다. 모든 클래스가 기본적으로 불변이며, 가변 프로퍼티를 만들려면 명시적으로 선언해야 한다.

#### Kotlin에서의 객체 생성

Kotlin의 생성자 결정 알고리즘은 다음과 같다(자바와 약간 다름에 주의).

1. `@PersistenceCreator`가 붙은 생성자가 있다면 그것을 쓴다.
2. 타입이 Kotlin `data class`이면 primary constructor를 쓴다.
3. `@PersistenceCreator`가 붙은 단 하나의 정적 팩토리 메서드가 있다면 그것을 쓴다.
4. 생성자가 단 하나라면 그것을 쓴다.
5. 생성자가 여러 개인데 정확히 하나에만 `@PersistenceCreator`가 붙어 있다면 그것을 쓴다.
6. 타입이 자바 `Record`이면 canonical constructor를 쓴다.
7. no-argument 생성자가 있다면 그것을 쓴다.

가장 단순한 Kotlin `data class` 예시.

```kotlin
data class Person(val id: String, val name: String)
```

`@PersistenceCreator`로 보조 생성자를 영속 생성자로 지정한 예시.

```kotlin
data class Person(var id: String, val name: String) {

    @PersistenceCreator
    constructor(id: String) : this(id, "unknown")
}
```

파라미터 기본값을 활용한 예시. 도큐먼트에 `name`이 없거나 `null`이면 `"unknown"`이 적용된다.

```kotlin
data class Person(var id: String, val name: String = "unknown")
```

> **Note**  
> Delegated property는 Spring Data에서 지원되지 않는다. Delegated property가 만들어내는 합성 필드는 `@Transient`로 제외해야 한다.

#### Kotlin Data Class의 프로퍼티 채우기

Kotlin data class는 사실상 불변이다.

```kotlin
data class Person(val id: String, val name: String)
```

Kotlin은 자동으로 `copy(…)` 메서드를 만들어주는데, 이 메서드가 모든 값을 복사하면서 인자로 전달된 값만 갈아끼운 새 인스턴스를 반환한다. Spring Data의 불변 객체용 매핑이 이와 잘 맞물린다.

#### Kotlin에서의 프로퍼티 오버라이딩

```kotlin
open class SuperType(open var field: Int)

class SubType(override var field: Int = 1) :
    SuperType(field) {
}
```

이 코드는 각 클래스마다 별도의 프로퍼티 접근자를 생성한다. 컴파일된 자바 코드는 다음과 같다.

```java
public class SuperType {

   private int field;

   public SuperType(int field) {
      this.field = field;
   }

   public int getField() {
      return this.field;
   }

   public void setField(int field) {
      this.field = field;
   }
}

public final class SubType extends SuperType {

   private int field;

   public SubType(int field) {
      super(field);
      this.field = field;
   }

   public int getField() {
      return this.field;
   }

   public void setField(int field) {
      this.field = field;
   }
}
```

`SubType`의 getter/setter는 `SubType.field`만 다룬다. 따라서 `SuperType.field`를 set하는 기본 경로는 생성자뿐이다. 자바 쪽 오버라이딩과 동일한 고려사항이 그대로 적용된다.

#### Kotlin Value Class

Kotlin Value Class는 더 표현력 있는 도메인 모델을 위한 도구다.

```kotlin
@JvmInline
value class EmailAddress(val theAddress: String)                                    // 1

data class Contact(val id: String, val name:String, val emailAddress: EmailAddress) // 2
```

1. non-nullable 값 타입을 가지는 단순한 value class.
2. 위 value class를 프로퍼티 타입으로 사용하는 data class.

non-nullable이며 non-primitive 값 타입을 사용하는 프로퍼티는 컴파일 시점에 value class가 평탄화되어 내부 값 타입으로 표현된다.

---

## 2. 컨벤션 기반 매핑 (Convention-based Mapping)

`MappingMongoConverter`는 별도 메타데이터가 없을 때 다음 컨벤션으로 객체와 도큐먼트를 매핑한다.

- 자바 클래스의 짧은 이름이 컬렉션 이름이 된다. 예: `com.bigbank.SavingsAccount` → 컬렉션 `savingsAccount`.
- 중첩된 객체는 DBRef가 아니라 도큐먼트 안의 중첩 객체로 저장된다.
- 등록된 Spring Converter가 있으면 그것이 기본 매핑보다 우선 적용된다.
- 자바 `JavaBean` 프로퍼티(getter/setter)가 아닌 **필드**를 변환의 기준으로 삼는다.
- non-zero-argument 생성자가 단 하나이고 그 인자 이름들이 도큐먼트 최상위 필드들과 일치하면 그 생성자가 쓰인다. 아니면 zero-argument 생성자가 쓰이고, non-zero-argument 생성자가 둘 이상이면 예외가 발생한다.

### 2.1 `_id` 필드 처리 방식

MongoDB의 모든 도큐먼트에는 `_id` 필드가 있어야 한다. 명시하지 않으면 드라이버가 자동 생성된 ObjectId를 부여한다. `_id`는 유니크해야 하고, 배열만 아니면 어떤 타입이든 사용할 수 있다.

자바 클래스의 어떤 필드가 `_id`로 매핑되는지의 규칙은 다음과 같다.

- `@Id` (`org.springframework.data.annotation.Id`)가 붙은 필드는 `_id`로 매핑된다. 단, `@Field`로 도큐먼트 필드 이름을 따로 지정하면 도큐먼트에 `_id`가 들어가지 않는다.
- 어노테이션은 없지만 이름이 `id`인 필드는 `_id`로 매핑된다.

#### `_id` 매핑 결과 예시

| 필드 정의                      | MongoDB 도큐먼트의 ID 필드명                   |
|----------------------------|----------------------------------------|
| `String id`                | `_id`                                  |
| `@Field String id`         | `_id`                                  |
| `@Field("x") String id`    | `x`                                    |
| `@Id String x`             | `_id`                                  |
| `@Field("x") @Id String y` | `_id` (`@Field(name)`는 무시되고 `@Id`가 우선) |

#### `_id` 필드의 타입 변환 규칙

- 자바 클래스의 `id` 필드가 `String` 또는 `BigInteger`라면, 가능한 경우 ObjectId로 변환되어 저장된다. ObjectId 타입을 직접 쓰는 것도 유효하다. 애플리케이션에서 `id` 값을 직접 지정하면 드라이버가 ObjectId로 변환을 시도하고, 변환할 수 없으면 그 값 그대로 `_id`에 저장된다. `@Id`가 붙어 있어도 마찬가지다.
- 필드에 `@MongoId`를 붙이면, 실제 자바 타입을 그대로 사용해서 변환·저장한다. `@MongoId`에서 별도로 타입을 지정하지 않는 한 추가 변환이 일어나지 않는다. 값이 없으면 새 `ObjectId`가 생성되어 프로퍼티 타입으로 변환된다.
- `@MongoId(FieldType.…)`처럼 타입을 명시하면, 그 `FieldType`으로 변환을 시도한다. 값이 없으면 새 `ObjectId`가 생성되어 선언된 타입으로 변환된다.
- `id` 필드의 타입이 `String`/`BigInteger`/`ObjectId`가 아니라면, 애플리케이션에서 직접 값을 부여해야 한다. 그래야 도큐먼트의 `_id`에 그대로 저장된다.
- 자바 클래스에 `id` 필드가 아예 없으면, 드라이버가 암묵적인 `_id`를 만들지만 자바 클래스의 어떤 프로퍼티에도 매핑되지 않는다.

쿼리/업데이트 시에도 `MongoTemplate`은 위 규칙에 따라 `Query`와 `Update` 객체를 컨버터로 변환한다. 그래서 쿼리에서는 도메인 클래스의 필드 이름과 타입을 그대로 쓰면 된다.

---

## 3. 데이터 매핑과 타입 변환 (Data Mapping and Type Conversion)

Spring Data MongoDB는 BSON으로 표현 가능한 모든 타입을 지원하고, 그 외의 타입을 위해서는 내장 컨버터를 제공한다. 사용자 정의 컨버터를 추가해 변환 규칙을 손볼 수도 있다.

### 3.1 내장 타입 변환 표

| Type                                     | Type conversion                                                  | Sample                                                                                                                                                                                      |
|------------------------------------------|------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `String`                                 | native                                                           | `{"firstname" : "Dave"}`                                                                                                                                                                    |
| `double`, `Double`, `float`, `Float`     | native                                                           | `{"weight" : 42.5}`                                                                                                                                                                         |
| `int`, `Integer`, `short`, `Short`       | native 32-bit integer                                            | `{"height" : 42}`                                                                                                                                                                           |
| `long`, `Long`                           | native 64-bit integer                                            | `{"height" : 42}`                                                                                                                                                                           |
| `Date`, `Timestamp`                      | native                                                           | `{"date" : ISODate("2019-11-12T23:00:00.809Z")}`                                                                                                                                            |
| `byte[]`                                 | native                                                           | `{"bin" : { "$binary" : "AQIDBA==", "$type" : "00" }}`                                                                                                                                      |
| `java.util.UUID` (UuidRepresentation 기준) | native                                                           | `{"uuid" : { "$binary" : "MEaf1CFQ6lSphaa3b9AtlA==", "$type" : "04" }}`                                                                                                                     |
| `Date`                                   | native                                                           | `{"date" : ISODate("2019-11-12T23:00:00.809Z")}`                                                                                                                                            |
| `ObjectId`                               | native                                                           | `{"_id" : ObjectId("5707a2690364aba3136ab870")}`                                                                                                                                            |
| Array, `List`, `BasicDBList`             | native                                                           | `{"cookies" : [ … ]}`                                                                                                                                                                       |
| `boolean`, `Boolean`                     | native                                                           | `{"active" : true}`                                                                                                                                                                         |
| `null`                                   | native                                                           | `{"value" : null}`                                                                                                                                                                          |
| `Document`                               | native                                                           | `{"value" : { … }}`                                                                                                                                                                         |
| `Decimal128`                             | native                                                           | `{"value" : NumberDecimal(…)}`                                                                                                                                                              |
| `AtomicInteger` (`get()` 후 변환)           | converter 32-bit integer                                         | `{"value" : "741" }`                                                                                                                                                                        |
| `AtomicLong` (`get()` 후 변환)              | converter 64-bit integer                                         | `{"value" : "741" }`                                                                                                                                                                        |
| `BigInteger`                             | native `NumberDecimal`, `String` (`BigDecimalRepresentation` 참조) | `{"value" : NumberDecimal(741) }`, `{"value" : "741" }`                                                                                                                                     |
| `BigDecimal`                             | native `NumberDecimal`, `String` (`BigDecimalRepresentation` 참조) | `{"value" : NumberDecimal(741.99) }`, `{"value" : "741.99" }`                                                                                                                               |
| `URL`                                    | converter                                                        | `{"website" : "https://spring.io/projects/spring-data-mongodb/" }`                                                                                                                          |
| `Locale`                                 | converter                                                        | `{"locale" : "en_US" }`                                                                                                                                                                     |
| `char`, `Character`                      | converter                                                        | `{"char" : "a" }`                                                                                                                                                                           |
| `NamedMongoScript`                       | converter `Code`                                                 | `{"_id" : "script name", value: (some javascript code)}`                                                                                                                                    |
| `java.util.Currency`                     | converter                                                        | `{"currencyCode" : "EUR"}`                                                                                                                                                                  |
| `Instant` (Java 8)                       | native                                                           | `{"date" : ISODate("2019-11-12T23:00:00.809Z")}`                                                                                                                                            |
| `Instant` (Java 8)                       | converter                                                        | `{"date" : ISODate("2019-11-12T23:00:00.809Z")}`                                                                                                                                            |
| `LocalDate` (Java 8)                     | converter / native (Java 8) [^1]                                 | `{"date" : ISODate("2019-11-12T00:00:00.000Z")}`                                                                                                                                            |
| `LocalDateTime`, `LocalTime` (Java 8)    | converter / native (Java 8) [^2]                                 | `{"date" : ISODate("2019-11-12T23:00:00.809Z")}`                                                                                                                                            |
| `ZoneId` (Java 8)                        | converter                                                        | `{"zoneId" : "ECT - Europe/Paris"}`                                                                                                                                                         |
| `Box`                                    | converter                                                        | `{"box" : { "first" : { "x" : 1.0 , "y" : 2.0} , "second" : { "x" : 3.0 , "y" : 4.0}}`                                                                                                      |
| `Polygon`                                | converter                                                        | `{"polygon" : { "points" : [ { "x" : 1.0 , "y" : 2.0} , { "x" : 3.0 , "y" : 4.0} , { "x" : 4.0 , "y" : 5.0}]}}`                                                                             |
| `Circle`                                 | converter                                                        | `{"circle" : { "center" : { "x" : 1.0 , "y" : 2.0} , "radius" : 3.0 , "metric" : "NEUTRAL"}}`                                                                                               |
| `Point`                                  | converter                                                        | `{"point" : { "x" : 1.0 , "y" : 2.0}}`                                                                                                                                                      |
| `GeoJsonPoint`                           | converter                                                        | `{"point" : { "type" : "Point" , "coordinates" : [3.0 , 4.0] }}`                                                                                                                            |
| `GeoJsonMultiPoint`                      | converter                                                        | `{"geoJsonLineString" : {"type":"MultiPoint", "coordinates": [ [ 0 , 0 ], [ 0 , 1 ], [ 1 , 1 ] ] }}`                                                                                        |
| `Sphere`                                 | converter                                                        | `{"sphere" : { "center" : { "x" : 1.0 , "y" : 2.0} , "radius" : 3.0 , "metric" : "NEUTRAL"}}`                                                                                               |
| `GeoJsonPolygon`                         | converter                                                        | `{"polygon" : { "type" : "Polygon", "coordinates" : [[ [ 0 , 0 ], [ 3 , 6 ], [ 6 , 1 ], [ 0 , 0 ] ]] }}`                                                                                    |
| `GeoJsonMultiPolygon`                    | converter                                                        | `{"geoJsonMultiPolygon" : { "type" : "MultiPolygon", "coordinates" : [ [ [ [ -73.958 , 40.8003 ] , [ -73.9498 , 40.7968 ] ] ], [ [ [ -73.973 , 40.7648 ] , [ -73.9588 , 40.8003 ] ] ] ] }}` |
| `GeoJsonLineString`                      | converter                                                        | `{ "geoJsonLineString" : { "type" : "LineString", "coordinates" : [ [ 40 , 5 ], [ 41 , 6 ] ] }}`                                                                                            |
| `GeoJsonMultiLineString`                 | converter                                                        | `{"geoJsonLineString" : { "type" : "MultiLineString", coordinates: [ [ [ -73.97162 , 40.78205 ], [ -73.96374 , 40.77715 ] ], [ [ -73.97880 , 40.77247 ], [ -73.97036 , 40.76811 ] ] ] }}`   |

### 3.2 Collection 처리

컬렉션 타입의 매핑 결과는 MongoDB가 실제로 돌려준 값에 따라 달라진다.

- 도큐먼트에 해당 필드 자체가 없으면, 매핑은 그 프로퍼티를 건드리지 않는다. 즉, 객체 생성 시점에 들어간 값(`null`, 자바 기본값, 생성자/팩토리에서 넣은 값 등)이 그대로 유지된다.
- 도큐먼트에 필드가 있는데 값이 `null`이라면 (`{ 'list' : null }`), 프로퍼티 값도 `null`로 설정된다.
- 도큐먼트에 필드가 있고 값이 `null`이 아니면 (`{ 'list' : [ … ] }`), 컬렉션이 매핑된 값들로 채워진다.

생성자 기반으로 생성하는 경우 빠진 값에 대한 기본값을 생성자/팩토리에서 부여할 수 있다. 프로퍼티 채우기 방식에서는 응답에 값이 없을 때 필드의 기본 초기값이 그대로 남는다.

---

## 4. 매핑 설정 (Mapping Configuration)

`MongoTemplate`을 만들 때 별도 설정이 없으면 `MappingMongoConverter`가 기본 인스턴스로 만들어진다. 직접 인스턴스화하면 ① 도메인 클래스를 어디에서 찾을지 지정해 메타데이터 추출과 인덱스 구성을 미리 할 수 있고, ② 특정 클래스에 대한 Spring 컨버터를 등록할 수도 있다.

`MappingMongoConverter`, `MongoClient`, `MongoTemplate`은 Java/XML 설정 모두로 구성할 수 있다.

### 4.1 Java 기반 설정

```java
@Configuration
public class MongoConfig extends AbstractMongoClientConfiguration {

  @Override
  public String getDatabaseName() {
    return "database";
  }

  // the following are optional

  @Override
  public String getMappingBasePackage() { // 1
    return "com.bigbank.domain";
  }

  @Override
  void configureConverters(MongoConverterConfigurationAdapter adapter) { // 2

    adapter.registerConverter(new org.springframework.data.mongodb.test.PersonReadConverter());
    adapter.registerConverter(new org.springframework.data.mongodb.test.PersonWriteConverter());
  }

  @Bean
  public LoggingEventListener<MongoMappingEvent> mappingEventsListener() {
    return new LoggingEventListener<MongoMappingEvent>();
  }
}
```

1. 매핑 기본 패키지는 `MappingContext`를 미리 초기화할 때 엔티티를 스캔할 루트 경로다. 기본값은 설정 클래스가 위치한 패키지다.
2. 특정 도메인 타입의 기본 매핑 절차를 사용자 정의 컨버터로 대체할 수 있다.

`AbstractMongoClientConfiguration`는 `MongoClient`와 데이터베이스 이름을 제공하는 메서드 구현을 요구한다. 더해서 `getMappingBasePackage(...)`를 오버라이드하면 컨버터에게 `@Document` 어노테이션이 붙은 클래스를 어디에서 스캔할지 알려줄 수 있다.

### 4.2 XML 기반 설정

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xmlns:mongo="http://www.springframework.org/schema/data/mongo"
  xsi:schemaLocation="
    http://www.springframework.org/schema/data/mongo https://www.springframework.org/schema/data/mongo/spring-mongo.xsd
    http://www.springframework.org/schema/beans https://www.springframework.org/schema/beans/spring-beans-3.0.xsd">

  <!-- Default bean name is 'mongo' -->
  <mongo:mongo-client host="localhost" port="27017"/>

  <mongo:db-factory dbname="database" mongo-ref="mongoClient"/>

  <!-- by default look for a Mongo object named 'mongo' - default name used for the converter is 'mappingConverter' -->
  <mongo:mapping-converter base-package="com.bigbank.domain">
    <mongo:custom-converters>
      <mongo:converter ref="readConverter"/>
      <mongo:converter>
        <bean class="org.springframework.data.mongodb.test.PersonWriteConverter"/>
      </mongo:converter>
    </mongo:custom-converters>
  </mongo:mapping-converter>

  <bean id="readConverter" class="org.springframework.data.mongodb.test.PersonReadConverter"/>

  <!-- set the mapping converter to be used by the MongoTemplate -->
  <bean id="mongoTemplate" class="org.springframework.data.mongodb.core.MongoTemplate">
    <constructor-arg name="mongoDbFactory" ref="mongoDbFactory"/>
    <constructor-arg name="mongoConverter" ref="mappingConverter"/>
  </bean>

  <bean class="org.springframework.data.mongodb.core.mapping.event.LoggingEventListener"/>

</beans>
```

XML의 `base-package` 속성은 `@Document` 어노테이션이 붙은 클래스를 스캔할 위치를 지정한다.

### 4.3 보충 사항

> **Tip — Java Time타입**  
> MongoDB의 네이티브 JSR-310 지원은 `MongoConverterConfigurationAdapter.useNativeDriverJavaTimeCodecs()`로 활성화할 수 있다. 이 방식은 UTC를 기반으로 동작하므로 권장된다. 기본 JSR-310 지원은 로컬 머신 타임존을 사용하므로 하위 호환을 위해서만 쓰는 것이 좋다. 위 예시의 `LoggingEventListener`는 Spring의 `ApplicationContextEvent` 인프라에 발행되는 `MongoMappingEvent` 인스턴스를 로깅한다.

> **Note**  
> `AbstractMongoClientConfiguration`는 `MongoTemplate` 인스턴스를 생성해 컨테이너에 `mongoTemplate`이라는 이름으로 등록한다.

> **Tip — Spring Boot와 함께 쓰기**  
> Spring Boot로 부트스트랩하면서 일부 설정만 덮어쓰고 싶다면, 해당 타입의 빈을 그대로 노출하면 된다. 예를 들어 커스텀 변환을 적용하려면 `MongoCustomConversions` 빈을 등록하면 Boot 인프라가 알아서 픽업해 준다. 자세한 내용은 [Spring Boot 레퍼런스 문서](https://docs.spring.io/spring-boot/reference/data/nosql.html#data.nosql.mongodb)를 참고하자.

---

## 5. 메타데이터 기반 매핑 (Metadata-based Mapping)

Spring Data MongoDB의 객체 매핑 기능을 제대로 활용하려면 매핑 대상 객체에 `@Document`를 붙여 두는 게 좋다. 매핑 프레임워크가 동작하는 데 이 어노테이션이 반드시 필요한 건 아니지만(어노테이션 없는 POJO도 매핑은 잘 된다), classpath 스캐너가 도메인 객체를 미리 찾아 메타데이터를 추출하게 해 준다. 이 어노테이션이 없으면 메타데이터가 처음 객체를 저장할 때 비로소 구축되어, 그 시점에 약간의 성능 손실이 생긴다.

```java
package com.mycompany.domain;

@Document
public class Person {

  @Id
  private ObjectId id;

  @Indexed
  private Integer ssn;

  private String firstName;

  @Indexed
  private String lastName;
}
```

> **Important**  
> `@Id`는 어떤 프로퍼티가 MongoDB의 `_id`로 쓰일지를 매퍼에게 알려주고, `@Indexed`는 매핑 프레임워크가 그 프로퍼티에 대해 `createIndex(…)`를 호출하도록 지시해 검색 속도를 높여 준다. 자동 인덱스 생성은 `@Document`가 붙은 타입에 대해서만 일어난다.

> **Warning**  
> 자동 인덱스 생성은 기본적으로 비활성화되어 있다. 설정에서 명시적으로 켜야 한다(Index Creation 섹션 참조).

### 5.1 매핑 어노테이션 개요

`MappingMongoConverter`가 메타데이터 기반 매핑에 사용하는 어노테이션 목록.

- **`@Id`** — 필드 레벨. 식별자 용도로 쓰인다.
- **`@MongoId`** — 필드 레벨. 식별자 용도. 선택적인 `FieldType`으로 ID 변환을 커스터마이징할 수 있다.
- **`@Document`** — 클래스 레벨. 이 클래스가 매핑 후보임을 표시한다. 저장될 컬렉션 이름을 지정할 수 있다.
- **`@DBRef`** — 필드 레벨. `com.mongodb.DBRef`로 저장됨을 표시한다.
- **`@DocumentReference`** — 필드 레벨. 다른 도큐먼트에 대한 포인터로 저장됨을 표시한다. 단일 값(기본은 ID)일 수도 있고, 컨버터를 통한 `Document`일 수도 있다.
- **`@Indexed`** — 필드 레벨. 인덱싱 방법을 기술한다.
- **`@CompoundIndex`** (반복 가능) — 타입 레벨. 복합 인덱스를 선언한다.
- **`@GeoSpatialIndexed`** — 필드 레벨. 지오 인덱스 방법을 기술한다.
- **`@TextIndexed`** — 필드 레벨. 텍스트 인덱스에 포함될 필드임을 표시한다.
- **`@HashIndexed`** — 필드 레벨. 샤딩된 클러스터에서 데이터를 분산하는 해시 인덱스에 사용된다.
- **`@Language`** — 필드 레벨. 텍스트 인덱스의 language 속성을 설정한다.
- **`@Transient`** — 기본적으로 모든 필드는 도큐먼트에 매핑되지만, 이 어노테이션이 붙은 필드는 저장에서 제외된다. 컨버터가 생성자 인자에 대한 값을 구할 수 없게 되므로, transient 프로퍼티는 persistence constructor에서 사용할 수 없다.
- **`@PersistenceCreator`** — 데이터베이스에서 객체를 인스턴스화할 때 사용할 생성자(또는 정적 팩토리 메서드)를 지정한다. package-private 생성자도 지정할 수 있다. 생성자 인자는 검색된 도큐먼트의 키 값에 이름으로 매핑된다.
- **`@Value`** — Spring Framework 표준 어노테이션. 매핑 프레임워크 안에서는 생성자 인자에 사용해, 도메인 객체 구성에 사용되기 전에 데이터베이스에서 가져온 값을 SpEL로 변환할 수 있다. 도큐먼트의 프로퍼티를 참조할 때는 `@Value("#root.myProperty")`와 같이 `root`를 통해 접근한다.
- **`@Field`** — 필드 레벨. BSON 도큐먼트에서 표현될 필드의 이름과 타입을 기술해, 자바 클래스의 필드명/타입과 다른 값을 사용할 수 있게 한다.
- **`@Version`** — 필드 레벨. 낙관적 잠금에 사용된다. 저장 시 수정 여부를 검사한다. 초기값은 0(primitive면 1)이고, 업데이트마다 자동 증가한다.

매핑 메타데이터 인프라는 기술 중립적인 별도 프로젝트인 `spring-data-commons`에 정의되어 있다. MongoDB 모듈은 어노테이션 기반 메타데이터를 지원하는 하위 클래스를 제공하지만, 원한다면 다른 전략도 직접 구현할 수 있다.

다음은 위 어노테이션들을 종합적으로 사용한 예시다.

```java
@Document
@CompoundIndex(name = "age_idx", def = "{'lastName': 1, 'age': -1}")
public class Person<T extends Address> {

  @Id
  private String id;

  @Indexed(unique = true)
  private Integer ssn;

  @Field("fName")
  private String firstName;

  @Indexed
  private String lastName;

  private Integer age;

  @Transient
  private Integer accountTotal;

  @DBRef
  private List<Account> accounts;

  private T address;

  public Person(Integer ssn) {
    this.ssn = ssn;
  }

  @PersistenceCreator
  public Person(Integer ssn, String firstName, String lastName, Integer age, T address) {
    this.ssn = ssn;
    this.firstName = firstName;
    this.lastName = lastName;
    this.age = age;
    this.address = address;
  }

  public String getId() {
    return id;
  }

  // no setter for Id.  (getter is only exposed for some unit testing)

  public Integer getSsn() {
    return ssn;
  }

// other getters/setters omitted
}
```

### 5.2 필드 타입 커스터마이징

매핑 인프라가 추론하는 MongoDB 네이티브 타입이 원하는 타입과 다를 때 `@Field(targetType=…)`이 유용하다. 예를 들어 구버전 MongoDB 서버는 `BigDecimal`을 직접 지원하지 않아 `Decimal128`이 아닌 `String`으로 표현되는 경우가 있다.

```java
public class Balance {

  @Field(targetType = STRING)
  private BigDecimal value;

  // ...
}
```

직접 커스텀 어노테이션을 만들어 더 깔끔하게 표현할 수도 있다.

```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
@Field(targetType = FieldType.STRING)
public @interface AsString { }

// ...

public class Balance {

  @AsString
  private BigDecimal value;

  // ...
}
```

### 5.3 특수한 필드 이름 (점이 포함된 키)

MongoDB는 점(`.`)을 중첩 도큐먼트나 배열의 경로 구분자로 사용한다. 즉 키 `a.b.c`는 다음 구조를 가리킨다.

```json
{
    "a" : {
        "b" : {
            "c" : …
        }
    }
}
```

MongoDB 5.0 이전에는 필드 이름에 점을 사용할 수 없었다. `Map`을 저장할 때 키에 점이 들어가는 문제를 우회하기 위해 `MappingMongoConverter#setMapKeyDotReplacement`로 치환자를 지정할 수 있었다.

```java
converter.setMapKeyDotReplacement("-");
// ...

source.map = Map.of("key.with.dot", "value")
converter.write(source,...) // -> map : { 'key-with-dot', 'value' }
```

MongoDB 5.0부터는 필드 이름에 점을 쓰는 제한이 풀렸다. `@Field`로 두 가지 의미를 구분할 수 있다.

1. **`@Field(name = "a.b")`** — 이름을 **경로**로 해석한다. 따라서 중첩된 도큐먼트 `{ a : { b : … } }`를 기대하게 된다.
2. **`@Field(name = "a.b", fieldNameType = KEY)`** — 이름을 **있는 그대로의 키**로 본다. 즉 `{ 'a.b' : … }`를 기대한다.

점이 포함된 필드 이름은 derived query method로 직접 타겟할 수 없다. 다음 `Item`의 `categoryId`는 `cat.id`라는 키로 저장된다.

```java
public class Item {

    @Field(name = "cat.id", fieldNameType = KEY)
    String categoryId;

    // ...
}
```

원시 표현은 다음과 같다.

```json
{
    "cat.id" : "5b28b5e7-52c2",
    ...
}
```

`cat.id` 필드를 직접 타겟할 수 없으므로, 이런 경우엔 [Aggregation Framework](https://docs.spring.io/spring-data/mongodb/reference/mongodb/aggregation-framework.html)를 사용해야 한다.

#### 점이 들어간 필드 조회 예시

```java
template.query(Item.class)
    // $expr : { $eq : [ { $getField : { input : '$$CURRENT', 'cat.id' }, '5b28b5e7-52c2' ] }
    .matching(expr(ComparisonOperators.valueOf(ObjectOperators.getValueOf("value")).equalToValue("5b28b5e7-52c2"))) // 1
    .all();
```

1. 매핑 레이어가 프로퍼티명 `value`를 실제 필드 이름으로 변환해 준다. 실제 필드 이름을 직접 써도 무방하다.

#### 점이 들어간 필드 업데이트 예시

```java
template.update(Item.class)
    .matching(where("id").is("r2d2"))
    // $replaceWith: { $setField : { input: '$$CURRENT', field : 'cat.id', value : 'af29-f87f4e933f97' } }
    .apply(AggregationUpdate.newUpdate(ReplaceWithOperation.replaceWithValue(ObjectOperators.setValueTo("value", "af29-f87f4e933f97")))) // 1
    .first();
```

1. 마찬가지로 매핑 레이어가 `value`를 실제 필드 이름으로 변환한다.

위 예시들은 최상위 도큐먼트의 단순한 케이스다. 중첩이 깊어질수록 필요한 aggregation 표현식의 복잡도도 함께 올라간다.

### 5.4 객체 생성 커스터마이징 (Customized Object Construction)

매핑 서브시스템은 생성자에 `@PersistenceCreator`를 붙여 객체 생성 방식을 커스터마이징할 수 있게 해 준다. 생성자 파라미터 값은 다음 순서로 결정된다.

- 파라미터에 `@Value`가 붙어 있으면, 그 표현식을 평가한 결과를 사용한다.
- 입력 도큐먼트에 동일한 이름의 필드가 있으면, 그 프로퍼티 정보를 사용한다.
- 둘 다 안 되면 `MappingException`이 발생한다.

```java
class OrderItem {

  private @Id String id;
  private int quantity;
  private double unitPrice;

  OrderItem(String id, @Value("#root.qty ?: 0") int quantity, double unitPrice) {
    this.id = id;
    this.quantity = quantity;
    this.unitPrice = unitPrice;
  }

  // getters/setters ommitted
}

Document input = new Document("id", "4711");
input.put("unitPrice", 2.5);
input.put("qty",5);
OrderItem item = converter.read(OrderItem.class, input);
```

`@Value` 안의 SpEL 표현식은 주어진 프로퍼티 경로가 풀리지 않을 경우 `0`을 fallback으로 사용한다.

### 5.5 매핑 프레임워크 이벤트

매핑 라이프사이클 동안 여러 이벤트가 발생한다. 해당 이벤트 빈을 Spring `ApplicationContext`에 등록해 두면, 이벤트가 디스패치될 때마다 호출된다. 자세한 내용은 [Lifecycle Events](https://docs.spring.io/spring-data/mongodb/reference/mongodb/lifecycle-events.html) 섹션을 참고하자.

---

[^1]: UTC 존 오프셋을 사용한다. `MongoConverterConfigurationAdapter`로 구성한다.
[^2]: UTC 존 오프셋을 사용한다. `MongoConverterConfigurationAdapter`로 구성한다.
