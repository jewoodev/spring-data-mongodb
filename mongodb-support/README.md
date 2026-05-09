> **원문 저작권:** Copyright © 2008-2025 VMware, Inc. (Broadcom Inc.). All rights reserved.
>
> 본 문서의 사본은 개인적인 용도 및 타인에게 배포하는 용도로 제작할 수 있습니다.
> 단, 사본에 대해 어떠한 수수료도 부과해서는 안 되며, 인쇄물이든 전자적 형태이든
> 배포하는 모든 사본에는 본 저작권 고지가 포함되어야 합니다.
>
> **원문:** <https://docs.spring.io/spring-data/mongodb/reference/mongodb.html>
>
> 본 문서는 학습 목적의 비공식 한국어 번역이며, 정확한 내용은 원문을 참조하시기 바랍니다.

# MongoDB Support

Spring Data가 제공하는 MongoDB 지원 기능은 폭넓은 범위를 가진다.

- Java 기반 `@Configuration` 클래스나 XML 네임스페이스를 활용하여 Mongo 드라이버 인스턴스 및 레플리카 셋을 구성할 수 있는 Spring 설정 지원.
- 자주 쓰는 Mongo 작업의 생산성을 높여주는 `MongoTemplate` 헬퍼 클래스. 문서(document)와 POJO 사이의 객체 매핑이 통합되어 있다.
- Spring의 일관된(portable) Data Access Exception 계층으로 변환되는 예외 변환 기능.
- Spring Conversion Service와 통합된 풍부한 객체 매핑.
- 다른 메타데이터 포맷도 지원할 수 있도록 확장 가능한, 어노테이션 기반의 매핑 메타데이터.
- 영속성(persistence) 및 매핑 라이프사이클 이벤트.
- Java 기반의 Query, Criteria, Update DSL.
- 사용자 정의 쿼리 메서드까지 포함하는 Repository 인터페이스의 자동 구현.
- 타입 세이프 쿼리를 위한 QueryDSL 통합.
- 다중 문서(Multi-Document) 트랜잭션.
- 지리 공간(GeoSpatial) 통합.
- 벡터 인덱스(Vector Index) 및 선언형 Vector Search 지원.
- AOT(Ahead Of Time) 최적화.

## MongoTemplate과 Repository 중에 무엇을 쓸 것인가
대부분의 작업에서는 `MongoTemplate` 또는 Repository 지원을 사용하는 것이 좋다. 둘 다 풍부한 매핑 기능을 활용할 수 있다.

`MongoTemplate`는 카운터 증가 같은 동작이나 ad-hoc CRUD 작업처럼, 직접적으로 어떤 기능을 호출해야 할 때 찾게되는 도구이다. 또한 `MongoTemplate`은 콜백 메서드를 제공하므로 `com.mongodb.client.MongoDatabase` 같은 저수준 API 객체를 손에 쥐고 MongoDB와 직접 통신하는 것이 어렵지 않다. 다양한 API 요소의 명명 규칙은 기본 MongoDB Java 드라이버의 이름을 그대로 따라가는 것을 목표로 하므로, 기존에 알고 있던 드라이버 지식을 Spring API에 그대로 옮겨 적용할 수 있다.

## 하위 주제
다음의 표는 [공식 문서의 MongoDB 페이지](https://docs.spring.io/spring-data/mongodb/reference/mongodb.html)가 다루는 하위 주제들을 정리한 것이다. 이 저장소에 번역본이 있는 항목은 링크가 걸려 있다.

| 주제                                                                                                                       | 설명                                                                              |
|--------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------|
| [Requirements](requirements.md)                                                                                          | Spring Data MongoDB가 요구하는 JDK / Spring Framework / MongoDB 드라이버 및 데이터베이스 버전.    |
| [Getting Started](https://docs.spring.io/spring-data/mongodb/reference/mongodb/getting-started.html)                     | 새로운 프로젝트에서 Spring Data MongoDB를 시작하는 방법.                                        |
| [Connecting to MongoDB](https://docs.spring.io/spring-data/mongodb/reference/mongodb/configuration.html)                 | `MongoClient` 와 `MongoDatabaseFactory` 등 MongoDB 연결 설정.                         |
| [Template API](template-api/README.md)                                                                                   | `MongoTemplate`/`MongoOperations` 사용법, fluent API, 예외 변환, 도메인 타입 매핑.            |
| [GridFS Support](https://docs.spring.io/spring-data/mongodb/reference/mongodb/template-gridfs.html)                      | 대용량 파일 저장을 위한 GridFS 지원.                                                        |
| [Object Mapping](./object-mapping/README.md)                                                                             | 객체와 BSON 문서를 매핑하는 방식, 어노테이션, 컨벤션 기반 매핑.                                         |
| [Value Expressions Fundamentals](https://docs.spring.io/spring-data/mongodb/reference/mongodb/value-expressions.html)    | 매핑 메타데이터 등에서 사용 가능한 SpEL/Property Placeholder 표현식.                              |
| [Lifecycle Events](https://docs.spring.io/spring-data/mongodb/reference/mongodb/lifecycle-events.html)                   | save/load 등 영속성 동작 전후로 발생하는 매핑 라이프사이클 이벤트.                                      |
| [Auditing](https://docs.spring.io/spring-data/mongodb/reference/mongodb/auditing.html)                                   | 작성자/수정자/시각 등 감사(auditing) 정보 자동 채움.                                             |
| [Sessions & Transactions](https://docs.spring.io/spring-data/mongodb/reference/mongodb/client-session-transactions.html) | 클라이언트 세션과 다중 문서 트랜잭션.                                                           |
| [Change Streams](https://docs.spring.io/spring-data/mongodb/reference/mongodb/change-streams.html)                       | 컬렉션 변경 사항을 스트리밍으로 구독하는 Change Streams.                                          |
| [Tailable Cursors](https://docs.spring.io/spring-data/mongodb/reference/mongodb/tailable-cursors.html)                   | capped collection을 대상으로 한 Tailable 커서.                                          |
| [Sharding](https://docs.spring.io/spring-data/mongodb/reference/mongodb/sharding.html)                                   | 샤딩 환경에서의 동작과 샤드 키 처리.                                                           |
| [MongoDB Search](https://docs.spring.io/spring-data/mongodb/reference/mongodb/mongo-search-indexes.html)                 | Atlas Search / Vector Search 인덱스 정의 및 사용.                                       |
| [Encryption](https://docs.spring.io/spring-data/mongodb/reference/mongodb/mongo-encryption.html)                         | 클라이언트 사이드 필드 레벨 암호화(Client-Side Field Level Encryption) / Queryable Encryption. |
| [Ahead of Time Optimizations](https://docs.spring.io/spring-data/mongodb/reference/mongodb/aot.html)                     | AOT 단계에서 적용 가능한 매핑/리포지토리 최적화.                                                   |
