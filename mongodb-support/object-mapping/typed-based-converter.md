> **원문 저작권:** Copyright © 2008-2025 VMware, Inc. (Broadcom Inc.). All rights reserved.
>
> 본 문서의 사본은 개인적인 용도 및 타인에게 배포하는 용도로 제작할 수 있습니다.
> 단, 사본에 대해 어떠한 수수료도 부과해서는 안 되며, 인쇄물이든 전자적 형태이든
> 배포하는 모든 사본에는 본 저작권 고지가 포함되어야 합니다.
>
> **원문:** <https://docs.spring.io/spring-data/mongodb/reference/mongodb/mapping/custom-conversions.html>
>
> 본 문서는 학습 목적의 비공식 한국어 번역이며, 정확한 내용은 원문을 참조하시기 바랍니다.

# Custom Conversions

도메인 객체와 MongoDB 문서 간의 매핑 결과를 직접 제어하고 싶을 때, Spring `Converter` 인터페이스를 구현해서 등록하면 된다. 다음은 String 값을 `Email`이라는 값 객체로 읽어 들이는 변환기 예시다.

```java
@ReadingConverter
public class EmailReadConverter implements Converter<String, Email> {

  public Email convert(String source) {
    return Email.valueOf(source);
  }
}
```

소스 타입과 대상 타입이 모두 네이티브 타입인 `Converter`를 작성한 경우, 인프라스트럭처 입장에서는 그것을 읽기 변환기로 다뤄야 할지 쓰기 변환기로 다뤄야 할지 자동으로 판단할 수 없다. 그런데 그런 변환기 인스턴스가 양쪽 방향 모두로 등록되어 버리면 의도치 않은 결과가 생길 수 있다. 가령 `Converter<String, Long>`은 모호한 변환기에 해당한다. 등록 자체는 가능하지만 쓰기 시점에 모든 String 값을 Long으로 바꿔 버리는 동작은 보통 원하는 결과가 아닐 것이다. 이런 모호함을 해소하고 한쪽 방향으로만 등록되도록 강제하기 위해 `@ReadingConverter`와 `@WritingConverter`라는 두 어노테이션이 변환기 구현체에 사용할 수 있는 형태로 제공된다.

변환기는 명시적으로 등록되어야 하는 인스턴스다. 클래스패스나 컨테이너 스캔으로 자동 수집되지 않는데, 의도하지 않은 변환기가 conversion service에 끼어들면서 생기는 부작용을 막기 위함이다. 변환기는 `CustomConversions`에 등록되며, 이 시설은 등록과 함께 소스/대상 타입 기준으로 변환기를 조회하는 역할도 담당하는 중앙 시설이다.

`CustomConversions`에는 사전 정의된 변환기들이 함께 포함되어 출고된다.

- `java.time` 패키지의 타입들과 `java.util.Date`, `String` 사이를 오가는 JSR-310 변환기

> _Note_
> 
> 로컬 시간 타입에 대한 기본 변환기들(예: `LocalDateTime`을 `java.util.Date`로 변환)은 시스템 기본 타임존 설정에 의존해 변환을 수행한다. 다른 기준이 필요하다면 사용자가 만든 변환기를 등록해서 기본 변환기를 덮어쓸 수 있다.

## Converter Disambiguation

기본적으로 Spring Data MongoDB는 `Converter` 구현체의 소스 타입과 대상 타입을 들여다본다. 그 중 한쪽이 데이터 액세스 API가 네이티브로 처리할 수 있는 타입인지를 판단해서 변환기를 읽기용 또는 쓰기용으로 등록한다.

```java
// 대상 타입만 네이티브로 처리 가능 -> 쓰기 컨버터로 등록된다
class MyConverter implements Converter<Person, String> { … }

// 소스 타입만 네이티브로 처리 가능 -> 읽기 컨버터로 등록된다
class MyConverter implements Converter<String, Person> { … }
```

## Type based Converter

매핑 결과에 영향을 주는 가장 손쉬운 방법은 `@Field` 어노테이션의 `targetType` 속성으로 MongoDB에 저장될 네이티브 타입을 지정해 두는 것이다. 이 방식을 쓰면 도메인 모델의 프로퍼티 타입이 `BigDecimal`처럼 MongoDB 네이티브 타입이 아니더라도, 저장 시점에는 `org.bson.types.Decimal128` 같은 네이티브 포맷으로 영속화하도록 만들 수 있다.

```java
public class Payment {

  @Id String id;  ➊

  @Field(targetType = FieldType.DECIMAL128)  ➋
  BigDecimal value;

  Date date;  ➌
}
```

```mongodb-json
{
  "_id"   : ObjectId("5ca4a34fa264a01503b36af8"), ➊ 
  "value" : NumberDecimal(2.099), ➋
  "date"  : ISODate("2019-04-03T12:11:01.870Z")  ➌
}
```

1. 유효한 `ObjectId` 형태의 String id는 자동으로 `ObjectId`로 변환되어 저장된다.
2. 저장될 타입을 명시적으로 지정해 두었다. 이 지정이 없다면 `BigDecimal`은 String으로 저장된다.
3. `Date` 값은 MongoDB 드라이버가 자체적으로 다뤄 `ISODate`로 저장한다.

이 정도의 단순한 타입 힌트만으로도 충분한 경우가 많다. 그러나 매핑 과정 자체를 더 세밀하게 손보고 싶다면, Spring `Converter` 구현체를 만들어 `MappingMongoConverter` 같은 `MongoConverter` 구현체에 함께 등록하는 방식을 쓸 수 있다.

`MappingMongoConverter`는 객체를 매핑하기 전에 해당 클래스를 다룰 수 있는 Spring 변환기가 등록되어 있는지를 먼저 확인한다. 그러므로 성능 최적화나 특수한 매핑 요구사항 때문에 `MappingMongoConverter`의 일반적인 매핑 전략을 우회하고 싶다면, Spring `Converter` 구현체를 먼저 만든 다음 그것을 `MappingConverter`에 등록해 주면 된다.

> _Note_
> 
> Spring 타입 변환 서비스 자체에 대한 더 자세한 내용은 [Spring 레퍼런스 문서](https://docs.spring.io/spring-framework/reference/core.html#validation)에서 확인할 수 있다.

## Writing Converter

쓰기 변환기는 도메인 객체를 MongoDB가 다룰 수 있는 형태(주로 `org.bson.Document`)로 직접 변환하는 역할을 한다. 다음은 `Person` 객체를 `Document`로 변환하는 예시다.

```java
import org.springframework.core.convert.converter.Converter;
import org.bson.Document;

public class PersonWriteConverter implements Converter<Person, Document> {

  public Document convert(Person source) {
    Document document = new Document();
    document.put("_id", source.getId());
    document.put("name", source.getFirstName());
    document.put("age", source.getAge());
    return document;
  }
}
```

이런 식으로 변환기를 직접 작성하면 어떤 필드를 어떤 키 이름으로 저장할지, 어떤 값을 빼고 넣을지 등을 코드 안에서 직접 결정할 수 있다.

## Reading Converter

읽기 변환기는 그 반대 방향이다. MongoDB의 `Document`를 도메인 객체로 되돌리는 역할을 한다.

```java
public class PersonReadConverter implements Converter<Document, Person> {

  public Person convert(Document source) {
    Person p = new Person((ObjectId) source.get("_id"), (String) source.get("name"));
    p.setAge((Integer) source.get("age"));
    return p;
  }
}
```

이 변환기 역시 BSON 문서에서 어떤 필드를 어떻게 꺼내 올지를 직접 코드에 적어 둔 형태다.

## Registering Converters

앞에서 언급했듯이 변환기는 자동으로 검색되지 않는다. 사용자가 직접 등록해 주어야 적용된다. 일반적인 방법은 `AbstractMongoClientConfiguration`을 상속한 설정 클래스에서 `configureConverters()` 메서드를 오버라이드해 변환기를 등록하는 것이다.

```java
class MyMongoConfiguration extends AbstractMongoClientConfiguration {

	@Override
	public String getDatabaseName() {
		return "database";
	}

	@Override
	protected void configureConverters(MongoConverterConfigurationAdapter adapter) {
		adapter.registerConverter(new com.example.PersonReadConverter());
		adapter.registerConverter(new com.example.PersonWriteConverter());
	}
}
```

`MongoConverterConfigurationAdapter`에 변환기를 추가해 두면 내부적으로 `CustomConversions`에 등록되어 매핑 시 사용된다.

## Big Number Format

MongoDB는 초기 버전에서 `BigDecimal` 같은 큰 수치 타입을 자체적으로 지원하지 않았다. 이런 사정 때문에 Spring Data MongoDB는 `BigDecimal`과 `BigInteger` 값을 저장할 때 String 표현으로 바꿔서 영속화하는 방식을 써왔다. 그런데 이 방식은 쿼리나 업데이트 시점에 숫자가 아닌 사전 순(lexical) 비교가 일어나는 등 여러 단점을 안고 있었다.

MongoDB Server 3.4부터는 `org.bson.types.Decimal128`이라는 네이티브 표현이 도입되어 `BigDecimal`과 `BigInteger`를 더 자연스럽게 다룰 수 있게 되었다. 이에 맞춰 **Spring Data MongoDB 5.0부터는 이 두 타입에 대한 기본 표현 방식이 더 이상 제공되지 않으며, 변환 방식을 명시적으로 설정해 주어야 한다.**

표현 방식은 여러 개를 함께 등록할 수 있고, 그 중 가장 먼저 등록된 것이 기본값으로 동작한다. 예를 들어 기존 동작과 동일하게 String을 기본 표현으로 두면서, 필요한 필드에서는 `Decimal128`도 쓸 수 있게 하고 싶다면 다음과 같이 `MongoCustomConversions`에서 `BigDecimalRepresentation`을 설정한다.

```java
MongoCustomConversions.create(config -> 
  config.bigDecimal(BigDecimalRepresentation.STRING, BigDecimalRepresentation.DECIMAL128)
)
```

이렇게 등록해 두면 기본 변환은 String으로 동작하면서도, 특정 필드에 `@Field(targetType = DECIMAL128)`을 붙이는 식으로 필드 단위로 저장 포맷을 따로 지정할 수 있다. 만약 프로젝트에서 `BigDecimal`/`BigInteger` 값을 실제로 저장하지 않는다면, 어느 표현 방식도 등록하지 않은 채 두는 것 또한 유효하다.

> _주의_
> 
> 매우 큰 값은 `BigDecimal` 또는 `BigInteger`로는 표현 가능한 범위 안에 있더라도, `org.bson.types.Decimal128`의 비트 길이 한계를 초과해서 네이티브 표현으로 저장하지 못할 수 있다는 점에 유의하자.
