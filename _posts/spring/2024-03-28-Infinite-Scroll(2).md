---
title: "QueryDSL, Slice를 사용한 no-offset 무한 스크롤 구현(2)"
categories:
    spring
tags:
    spring
    DB
date:
    2024-03-28 01:10:50+0900
toc:
    true
toc_sticky:
    true
---

(1)번 게시글에서 QueryDSL을 도입하기 위한 설정을 완료했다.
이제 실제로 QueryDSL을 이용한 무한 스크롤을 구현하기 위해 repository를 구성해야 한다.


# Repository Interface
``` java
public interface ProductRepository extends JpaRepository<Product, Long>{

}
```
기존 JPA Repository에 QueryDSL을 적용할 `ProductCustom` 인터페이스를 상속받도록 한다.
<br><br><br><br>

```java

public interface ProductCustom {

	Slice<Product> findAll(ProductListRequestDto requestDto);

}
```
QueryDSL을 사용할 Repository를 추상화하는 interface를 구성한다.
Slice를 사용할 것이기에 findAll의 return 값을 slice로 지정했다.<br><br><br><br><br>


# 실제 구현체
```java
@Repository
@RequiredArgsConstructor
public class ProductCustomImpl implements ProductCustom {
	private final JPAQueryFactory query;
	private final QProduct product = QProduct.product;

	@Override
	public Slice<Product> findAll(ProductListRequestDto requestDto) {
		Pageable pageable = PageRequest.of(1, requestDto.getPageSize());
		List<Product> results = query
			.selectFrom(product)
			.where(ltProductId(requestDto.getLastId()))
			.where(catCode(requestDto.getCategoryCode()))
			.orderBy(product.id.desc())
			.limit(requestDto.getPageSize()+2)
			.fetch();

		return checkLastPage(pageable, results);
	}

	
	// categoryCode 있는지 확인
	private BooleanExpression catCode(Integer categoryCode){
		if (categoryCode == null) {
			return null;
		}
		return product.category.code.eq(categoryCode);
	}

	
	// 다음 페이지 있는지 확인
	private BooleanExpression ltProductId(Long productId) {
		if (productId == null) {
			return null;
		}
		return product.id.lt(productId);
	}

	// 무한 스크롤 방식 처리하는 메서드
	private Slice<Product> checkLastPage(Pageable pageable, List<Product> results) {

		boolean hasNext = false;

		// 조회한 결과 개수가 요청한 페이지 사이즈보다 크면 뒤에 더 있음, next = true
		if (results.size() > pageable.getPageSize()+1) {
			System.out.println("result size: "+results.size());
			hasNext = true;
			results.remove(pageable.getPageSize());
		}

		return new SliceImpl<>(results, pageable, hasNext);
	}
}
```

구현하고자 했던 요구사항은 다음과 같다.
> 카테고리를 코드로 받아서 해당하는 카테고리의 리스트를 출력한다.
> 카테고리 코드가 없다면 전체 리스트를 출려한다.
> 처음 요청의 경우 lastId가 없으므로 내림차순으로 처음부터 조회한다.

그렇다면 이러한 요구사항을 구현하기 위한 코드를 하나하나 뜯어보자.

```java
    @Override
	public Slice<Product> findAll(ProductListRequestDto requestDto) {
		Pageable pageable = PageRequest.of(1, requestDto.getPageSize());
		List<Product> results = query
			.selectFrom(product)
			.where(ltProductId(requestDto.getLastId()))
			.where(catCode(requestDto.getCategoryCode()))
			.orderBy(product.id.desc())
			.limit(requestDto.getPageSize()+2)
			.fetch();

		return checkLastPage(pageable, results);
	}
```
실제로 작동하는 findAll 코드이다. 
`ProductListRequestDto`에는 다음과 같은 값이 들어있다.
- int categoryCode -> 카테고리를 나타내는 commonCode 값
- long productId -> 마지막으로 조회한 id값(index)
- int pageSize -> 조회하고자 하는 페이지의 크기

queryDSL을 사용하여 product 테이블을 조회하였으며 클러스터링 인덱스인 id를 기준으로 조회하고 있다.
추가 페이지 여부를 확인하기 위해 `pageSize+2` 만큼 조회한다.
- 왜 1이 아니라 2를 더한 것인가?
  - 1을 더한만큼 조회하고 추가 페이지 여부를 확인하기 위한 마지막 요소를 삭제했더니 페이지 수가 `pageSize`보다 1 적게 출력됐다. 해당 문제를 위해 +2만큼 조회하고 nextPage가 존재하면 마지막 원소를 삭제해 줬다. 

pageSize만큼만 조회하는 것을 확인할 수 있는데 여기서 집중해야 할 것은 `.where(ltProductId(requestDto.getLastId()))` 절이다.
`ltProductId()` 메소드를 보면
```java
private BooleanExpression ltProductId(Long productId) {
		if (productId == null) {
			return null;
		}
		return product.id.lt(productId);
	}
```
사용자로부터 입력받은 id가 없다면 null을 반환하고 있다면 `product.id.lt(productId)`를 반환한다. 
여기서 lt는 `<`를 뜻하는 메소드로  productId보다 작은 값을 탐색하겠다는 의미가 된다.

**BooleanExpression이란?**<br>
동적쿼리를 위해 사용되는 클래스로 null이 반환되면 조건절에서 자동으로 제거된다.
{: .notice--info}

쿼리는 완성했으니 Slice 객체에 쓰일 다음 페이지 여부를 반환할 함수를 만든다.
```java
    private Slice<Product> checkLastPage(Pageable pageable, List<Product> results) {

            boolean hasNext = false;

            // 조회한 결과 개수가 요청한 페이지 사이즈보다 크면 뒤에 더 있음, next = true
            if (results.size() > pageable.getPageSize()+1) {
                System.out.println("result size: "+results.size());
                hasNext = true;
                results.remove(pageable.getPageSize());
            }

            return new SliceImpl<>(results, pageable, hasNext);
        }
```

2개 더 조회한 원소가 실제 페이지 사이즈 +1보다 크다면 남은 원소가 존재하는 것이므로 `hasNext`를 True로 반환해 준다.

> 그런데 적으면서 생각해보니 클라이언트로부터 받은 `pageSize`를 Service 단에서 1 더해주고 했으면 해결됐을 문제인 것 같다.

# 결론
이렇게 해서 무한 스크롤을 구현할 수 있었다. 사실 깊게 공부하지 않고 부딪히며 짠 코드라 QueryDSL도 제대로 안 쓰고 `Slice`, `Page`를 제대로 공부하지 않고 한 것이라 좀 아쉬움이 남는다. 
## 의문
- dto에 있는 값이 pageable에 들어가는 값이랑 겹치는 것 같다. 그럼 그냥 dto 대신 pageable로 받았으면 되는 것 아닐까?
- 완성된 코드를 보니 내 코드에선 `Slice`가 의미없는 존재가 된 것 같다.
