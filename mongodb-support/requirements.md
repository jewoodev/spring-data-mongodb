> **원문 저작권:** Copyright © 2008-2025 VMware, Inc. (Broadcom Inc.). All rights reserved.
>
> 본 문서의 사본은 개인적인 용도 및 타인에게 배포하는 용도로 제작할 수 있습니다.
> 단, 사본에 대해 어떠한 수수료도 부과해서는 안 되며, 인쇄물이든 전자적 형태이든
> 배포하는 모든 사본에는 본 저작권 고지가 포함되어야 합니다.
>
> **원문:** <https://docs.spring.io/spring-data/mongodb/reference/preface.html>
>
> 본 문서는 학습 목적의 비공식 한국어 번역이며, 정확한 내용은 원문을 참조하시기 바랍니다.

Spring Data MongoDB 4.x는 JDK 17 그 이상과 Spring Framework 6.2.8 그 이상을 필요로 한다.

데이터베이스 및 드라이버 측면에서는 MongoDB 버전 4.x 이상과 호환되는 MongoDB Java 드라이버(5.2.x)가 필요하다.

## 기타 호환성 표
다음의 표는 Spring Data 버전과 호환되는 MongoDB 드라이버/데이터베이스 버전을 요약한 것이다. 데이터베이스 버전은 Spring Data test suite를 통과한 세대이다. 애플리케이션에서 MongoDB 서버 변경 사항의 영향을 받는 기능을 사용하지 않는 한 최신 서버 버전을 사용할 수 있다. 드라이버 및 서버 버전 호환성은 [공식 MongoDB 드라이버 호환성 표](https://www.mongodb.com/docs/drivers/java/sync/current/compatibility/)를 참조하자.

| Spring Data Release Train | Spring Data MongoDB | Driver Version                 | Database Versions |
|---------------------------|---------------------|--------------------------------|-------------------|
| 2025.0                    | 4.5.x               | 5.3.x                          | 6.x to 8.x        |
| 2024.1                    | 4.4.x               | 5.2.x                          | 4.4.x to 8.x      |
| 2024.0                    | 4.3.x               | 4.11.x & 5.x                   | 4.4.x to 7.x      |
| 2023.1                    | 4.2.x               | 4.9.x                          | 4.4.x to 7.x      |
| 2023.0 (*)                | 4.1.x               | 4.9.x                          | 4.4.x to 6.x      |
| 2022.0 (*)                | 4.0.x               | 4.7.x                          | 4.4.x to 6.x      |
| 2021.2 (*)                | 3.4.x               | 4.6.x                          | 4.4.x to 5.0.x    |
| 2021.1 (*)                | 3.3.x               | 4.4.x                          | 4.4.x to 5.0.x    |
| 2021.0 (*)                | 3.2.x               | 4.1.x                          | 4.4.x             |
| 2020.0 (*)                | 3.1.x               | 4.1.x                          | 4.4.x             |
| Neumann (*)               | 3.0.x               | 4.0.x                          | 4.4.x             |
| Moore (*)                 | 2.2.x               | 3.11.x/Reactive Streams 1.12.x | 4.2.x             |
| Lovelace (*)              | 2.1.x               | 3.8.x/Reactive Streams 1.9.x   | 4.0.x             |

(*) End of OSS Support