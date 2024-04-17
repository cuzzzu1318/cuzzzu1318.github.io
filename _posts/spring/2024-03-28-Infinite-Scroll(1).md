---
title: "QueryDSL, Slice를 사용한 no-offset 무한 스크롤 구현(1)"
categories:
    spring
tags:
    spring
    DB
date:
    2024-03-28 00:56:52+0900
toc:
    true
toc_sticky:
    true
---

SSAFY 프로젝트를 앱으로 진행하다 보니 리스트를 불러올 때 페이지보다는 무한스크롤을 구현하고 싶다는 욕심이 생겼다.
기존 페이지네이션도 구현해본 적 없는 상황에서 어떻게 구현하면 좋을지 찾아보다 성능적으로 우수한 no-offset 페이지네이션을 발견하게 되어 구현하였다.
해당 방식을 구현하는 방식을 살펴보니 동적 쿼리를 위해 QueryDSL을 사용하게 되어 QueryDSL을 먼저 간단히 정리해 보려 한다!



# QueryDSL?

> 하이버네이트 쿼리 언어(HQL: Hibernate Query Language)의 쿼리를 타입에 안전하게 생성 및 관리해주는 프레임워크


- Spring Boot, JPA를 사용할 때 복잡한 쿼리, 동적 쿼리를 구현하는 데 한계가 있다. 이러한 문제를 해결할 수 있는 것이 바로 QueryDSL

# 목적

- JPQL 생성용 데이터 세팅

# 장점

- 문자가 아닌 코드로 쿼리를 작성할 수 있어 컴파일 시점에 문법 오류를 확인할 수 있다.
- 인텔리제이와 같은 IDE의 자동 완성 기능의 도움을 받을 수 있다.
- 복잡한 쿼리나 동적 쿼리 작성이 편리하다.
- 쿼리 작성 시 제약 조건 등을 메서드 추출을 통해 재사용할 수 있다.
- JPQL 문법과 유사한 형태로 작성할 수 있어 쉽게 적응할 수 있다.

# 단점

- JPA가 지원하는 표준 기술이 아니다보니 설정하는 과정이 복잡하다.
- 

# 사용

1. Dependency 설치

```java
dependencies {
    implementation 'com.querydsl:querydsl-jpa:5.0.0:jakarta'
    annotationProcessor "com.querydsl:querydsl-apt:5.0.0:jakarta"
    annotationProcessor "jakarta.annotation:jakarta.annotation-api"
    annotationProcessor "jakarta.persistence:jakarta.persistence-api"
}
```

1. Configuration QueryDSL
- 프로젝트 어디서나 사용할 수 있도록 클래스를 작성하여 `Bean`으로 만든다.
- EntityManager를 주입하여 생성

```java
@Configuration
@RequiredArgsConstructor
public class QueryDSLConfig {
    private final EntityManager entityManager;

    @Bean
    public JPAQueryFactory jpaQueryFactory() {
        return new JPAQueryFactory(entityManager);
    }
}
```

1. 사용

```java
@Repository
@RequiredArgsConstructor
public class MemberCustomRepositoryImpl implements MemberCustomRepository {
    private final JPAQueryFactory query;

    @Override
    public boolean existsByMemberEmail(String email) {
        return query.selectOne()
                .from(member)
                .where(member.email.eq(email))
                .fetchFirst() != null;
    }
}
```

# QClass

> QueryDSL이 컴파일 단계에서 생성하는 클래스
> 
- 엔티티의 메타 정보를 담고 있는 클래스.
- 앤티티와 형태가 같은 Static Class로 엔티티의 속성을 나타냄.

## 사용 이유

- 속성을 static 방식으로 표현하므로 IDE의 자동완성 사용 가능.
- 컴파일러가 타임을 확인할 수 있기 때문에 타입 관련 오류를 사전에 방지할 수 있다.

# 참고

[https://ittrue.tistory.com/292](https://ittrue.tistory.com/292)

[https://medium.com/mo-zza/spring-data-jpa-querydsl-적용-22a0364cd579](https://medium.com/mo-zza/spring-data-jpa-querydsl-%EC%A0%81%EC%9A%A9-22a0364cd579)

[https://velog.io/@jmjmjmz732002/SpringBoot-QueryDSL-JPA-1-사용-이유-생각해보기](https://velog.io/@jmjmjmz732002/SpringBoot-QueryDSL-JPA-1-%EC%82%AC%EC%9A%A9-%EC%9D%B4%EC%9C%A0-%EC%83%9D%EA%B0%81%ED%95%B4%EB%B3%B4%EA%B8%B0)

[QueryDSL이란?](https://lordofkangs.tistory.com/456)

