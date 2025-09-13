---
title: "📚 Frontend 개발자의 Pagination 생각 확장하기"
date: 2025-02-27 17:30:00 +0900
categories: [Web Development, Frontend, Backend, Pagination]
tags: [pagination, frontend, backend, cursor-pagination, offset-pagination] 
pin: false
toc: true
---


웹/앱 개발에서 필수적인 **페이지네이션(Pagination)** 은 대량의 데이터를 효율적으로 관리하기 위한 핵심 기술입니다. 
Frontend 개발자인 저는, Offset 방식과 Cursor 방식의 내부 동작 방식 차이를 깊이 고민하며 개발해 본 경험은 없었습니다. 
최근 **GraphQL**과 **Apollo Client** 프로젝트에서 **Cursor** 방식 페이지네이션(Pagination) 을 처음으로 본격적으로 사용하게 되었고, 
이번 글에서는 각 방식에 대한 Frontend 개발자 입장에서의 다양한 스택에서의 사용법을 제공하기 위해, Cursor 방식 예시는 **GraphQL + Apollo Client** 환경에서, 
**Offset** 방식 예시는 일반적인 Fetch API 또는 Axios를 사용하여 코드 예시를 작성해 보겠습니다. 
이를 통해 각 기술 스택별 페이지네이션 구현 방식과 차이점을 보여드리고자 합니다.

---

## 1. 페이지네이션의 필요성과 기본 개념

### 📌 왜 페이지네이션이 필요할까요?
- **웹사이트 성능 향상 🚀**: 대량 데이터 로딩으로 인한 웹사이트/앱 성능 저하 방지 (사용자 이탈 방지! 🏃‍♀️💨), 페이지 단위로 불러오면 빠르게 응답할 수 있습니다.
- **사용자 경험 개선 😊**: 정보 과부하 방지, 사용자가 한눈에 보기 쉽게 데이터를 나누어 제공하여 정보 탐색이 편리합니다. (UX 디자인의 기본)
- **서버 부하 감소(=데이터베이스 효율) 💾**: 한 번에 적은 데이터만 처리하므로 서버의 부담이 줄어듭니다.(쿼리 성능 향상, 데이터베이스 부하 감소)


### 📝 페이지네이션 방식 개요

| 특징             | Offset-based Pagination  | Cursor Pagination                 |
|-----------------|---------------------------------|------------------------------------|
| **Pagination 방식** | 🔢 Offset & Limit 기반           | 📍 Cursor 기반                      |
| **페이지 탐색**    | ➡️ 페이지 번호 직접 이동 용이       | ➡️ 순차적 탐색에 최적화 ("다음/이전")     |
| **구현 난이도**    | 🧱 비교적 쉬움                     | 🧱 약간 더 복잡 (초기 설정 및 cursor 관리)     |
| **대용량 데이터 성능** | 🐌 Offset 증가 시 성능 저하 가능성 높음 | 🚀 성능 우수, 대용량 데이터에 강점         |
| **데이터 일관성**   | ⚠️ 데이터 변경 시 불일치 가능성 존재   | 👍 데이터 일관성 높음 (특히 실시간 데이터)     |
| **총 Item Count** | ✅ 총 아이템 수 계산 필요 (별도 쿼리)  | Optional (필수 아님, 성능 최적화 가능)   |
| **주요 사용처**    | 🏢 일반적인 목록, 관리자 페이지, 검색 결과 | 📱 소셜 미디어 피드, 무한 스크롤, 실시간 스트리밍 |
| **개발 팁**       | - Indexing 최적화 (성능 개선)       | - Cursor 컬럼 신중한 선택 (고유성, 안정성) |
|                   | - 캐싱 전략 활용 (DB 부하 감소)       | - Cursor 암호화/보안 고려 (민감 정보 보호) |


---

## 2. Offset Pagination (페이지 번호 방식) 심화 분석

### 2.1 기본 개념 및 동작 원리
**Offset Pagination** 은 우리가 흔히 접하는 "페이지 번호" 방식을 그대로 사용합니다.  
예를 들어, 한 페이지에 10개의 데이터를 보여줄 때, 3페이지는 **OFFSET = (3 - 1) * 10 = 20** 으로 계산하여 데이터를 불러옵니다.  
프론트엔드에서는 사용자가 "3페이지" 버튼을 클릭하면 해당 페이지의 데이터를 요청하고, 백엔드에서는 SQL의 `LIMIT`과 `OFFSET`을 활용하여 데이터를 조회합니다.


### 2.2 백엔드 SQL 예제

#### 예제 1: 기본 SQL 쿼리 (고정 OFFSET)
```sql
-- products 테이블에서 3번째 페이지(한 페이지에 10개)를 조회하는 예제
SELECT id, name, created_at
FROM products
ORDER BY id ASC
LIMIT 10 OFFSET 20;
```
> **설명:**
> - `LIMIT 10`: 한 페이지당 10개의 데이터를 조회
> - `OFFSET 20`: (3 - 1) * 10 = 20 → 21번째 데이터부터 가져옴  
    

#### 예제 2: 동적 OFFSET 계산 SQL 쿼리
```mysql
-- ? placeholder 를 사용하여 동적으로 OFFSET 계산 (MySQL Prepared Statement)
-- Parameterized Query: ? placeholder 를 사용하여 SQL Injection 공격을 방지
SELECT id, name, created_at
FROM products
ORDER BY id ASC
LIMIT ?   --  페이지당 아이템 수
OFFSET ((? - 1) * ?);  -- 시작 Offset
```
> **설명:**  
> 사용자가 요청한 페이지 번호와 페이지 크기에 따라 OFFSET이 자동 계산됩니다.

#### 예제 3: 전체 아이템 수 조회 (페이지 수 계산용)
```sql
SELECT COUNT(*) FROM products;
```
> **설명:**  
> 전체 데이터 개수를 조회하여, 총 페이지 수 등을 계산할 때 사용됩니다.

### 2.3 프론트엔드 JavaScript 예제

#### 예제 4: fetch API를 이용한 페이지별 데이터 요청
```javascript
const fetchProductsByPage = async (page, pageSize) => {
  try {
    const params = new URLSearchParams({
      limit: pageSize,
      offset: (page - 1) * pageSize,  // offset 계산을 여기서 수행
    });

    const response = await fetch(`/api/products?${params.toString()}`);

    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    }
    
    const data = await response.json();
    
    console.log(`Products on page ${page}:`, data.products);
    console.log("Total product count:", data.totalCount);
    return data;
  } catch (error) {
    console.error("Error fetching products:", error);
    return { products: [], totalCount: 0 }; // 더 적절한 빈 객체 반환.
  }
};

// 예시: 2페이지, 한 페이지당 10개 아이템
const products = await fetchProductsByPage(2, 10);
```
> **설명:**
> - `offset` 계산으로 원하는 페이지의 시작점을 구합니다.
> - API 요청 후 받은 데이터를 화면에 출력합니다.  


### 2.4 실제 데이터 예시 (Mock Data)
#### 예제 5: 💾 Offset Pagination Mock Data & SQL 쿼리 실행 예시

**Mock Data (products 테이블, MySQL)**:

```mysql
-- products 테이블 생성 및 Mock Data 삽입 SQL (MySQL)
CREATE TABLE products (
      id INT AUTO_INCREMENT PRIMARY KEY,
      name VARCHAR(255) NOT NULL,
      price INTEGER NOT NULL,
      created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO products (name, price) VALUES
    ('Product A', 10), ('Product B', 20), ('Product C', 15), ('Product D', 25), ('Product E', 30),
    ('Product F', 18), ('Product G', 22), ('Product H', 12), ('Product I', 28), ('Product J', 35),
    ('Product K', 17), ('Product L', 21), ('Product M', 13), ('Product N', 29), ('Product O', 32),
    ('Product P', 19), ('Product Q', 26), ('Product R', 14), ('Product S', 31), ('Product T', 23);
```

-- ✅ Offset Pagination 쿼리 (실행 예시)
```mysql
SELECT id, name, price, created_at
FROM products
ORDER BY id ASC
LIMIT 5   -- 페이지당 5개
OFFSET 5  -- 2페이지 시작점 ( (2-1) * 5 = 5 )
;
```
| id   | name         | price | created_at          |
|------|--------------|-------|----------------------|
| 6    | Product F    | 18    | 2025-02-27 15:32:49 |
| 7    | Product G    | 22    | 2025-02-27 15:32:49 |
| 8    | Product H    | 12    | 2025-02-27 15:32:49 |
| 9    | Product I    | 28    | 2025-02-27 15:32:49 |
| 10   | Product J    | 35    | 2025-02-27 15:32:49 |

-- ✅ 총 아이템 수 쿼리 (실행 예시)
SELECT COUNT(*) FROM products;

| COUNT(*) |
|----------|
| 20 |


### 2.5 장단점
#### 👍 장점 

* **🧱 구현 용이성**:  SQL 쿼리 작성이 간단하고, 프레임워크/ORM 지원도 풍부하여 개발 생산성 향상 (빠른 개발, 유지보수 용이)
* **🧭 직관적인 페이지 탐색**:  페이지 번호를 통해 특정 페이지로 바로 이동하는 기능 구현이 쉬움 (사용자 편의성, 관리자 기능에 적합)
* **🛠️ 디버깅 용이**:  페이지 번호, OFFSET 값을 통해 특정 페이지 데이터 확인 및 문제 분석 용이 (개발/테스트 효율 증대)

#### 👎 단점

* **🐌 성능 문제 (Deep Offset)**:  OFFSET 값이 커질수록 쿼리 성능이 급격히 저하될 수 있음 (Full Table Scan 발생 가능성, 특히 대용량 데이터에서 심각)
* **⚠️ 데이터 불일치 (Data Race)**:  페이지 이동 중 데이터 삽입/삭제 발생 시, 페이지 데이터 중복 또는 누락 문제 발생 가능성 (특히 실시간 데이터 변경이 잦은 경우)
* **📊 총 아이템 수 쿼리**:  총 페이지 수 계산을 위해 별도의 `COUNT(*)` 쿼리 필요 (DB 부하 증가, 약간의 성능 오버헤드)

### 2.6 ⚠️단점: Offset Pagination, 왜 데이터 변경에 취약할까?

#### 1. Offset Pagination 기본 작동 원리 (책 목록 비유)
Offset 방식 페이지네이션은 마치 도서관에서 **"목록의 특정 순번부터 몇 권의 책을 가져와 주세요!"** 라고 요청하는 것과 같습니다.

* **요청 예시:** "목록에서 **11번째부터 10권** 보여줘"
* **시스템:** `OFFSET 10` (처음 10권 건너뛰기)과 `LIMIT 10`을 적용해서, 목록 상 **11번부터 20번** 책을 선택

여기서 핵심은 `OFFSET`이 **데이터 목록의 순번 (절대적인 위치)** 에 의존한다는 점입니다.

#### 2. 데이터 삭제 시나리오: 사라진 책, 꼬여버린 페이지 😵

여러분이 2페이지 (11번부터 20번 책)를 신나게 읽고 있는데, 갑자기 도서관 **1페이지 (1번부터 10번 책) 에서 책 3권이 사라진 겁니다** (데이터베이스에서 데이터 삭제 발생)

**📚 책장 상황 변화:**

* **삭제 전:**  1번, 2번, 3번, 4번, 5번, 6번, 7번, 8번, 9번, 10번, **11번**, 12번, 13번, 14번, 15번, 16번, 17번, 18번, 19번, 20번, ...
* **삭제 후:**  1번, 2번, **4번**, **6번**, **8번**, 9번, 10번, **기존 11번** (이제 8번!), **기존 12번** (이제 9번!), **기존 13번** (이제 10번!), **기존 14번** (이제 11번!), ... , **기존 20번** (이제 17번!), ...

**⚠️ 문제 발생:**

* 원래 2페이지의 **11번, 12번, 13번** 책들이 **앞쪽 책이 삭제되면서 순번이 꼬여버립니다.**
* 다음에 **같은 `OFFSET 10`, `LIMIT 10`** 으로 2페이지를 다시 요청하면, 시스템은 이제 **새롭게 11번부터 20번** 책을 가져오게 됩니다.
* 하지만 이 "새로운 2페이지" 는 **삭제 전 2페이지의 뒷부분 데이터 (원래 11번 이후 책들)** 와 **중복되거나, 아예 엉뚱한 데이터**를 보여주게 될 수 있습니다.

#### 3. ➕ 데이터 추가 시나리오: 늘어난 책, 밀려버린 페이지 🤯

이번에는 반대로 2페이지를 보고 있는데, **1페이지 앞에 새 책 3권이 "뿅!" 하고 나타났어요** (데이터베이스에 데이터 추가 발생)

**📚 책장 상황 변화:**

* **추가 전:** 1번, 2번, 3번, 4번, 5번, 6번, 7번, 8번, 9번, 10번, **11번**, 12번, 13번, 14번, 15번, 16번, 17번, 18번, 19번, 20번, ...
* **추가 후:**  **새로운 책 A**, **새로운 책 B**, **새로운 책 C**, 1번, 2번, 3번, 4번, 5번, 6번, 7번, 8번, 9번, 10번, **기존 11번** (이제 14번!), **기존 12번** (이제 15번!), **기존 13번** (이제 16번!), **기존 14번** (이제 17번!), ... , **기존 20번** (이제 23번!), ...

**⚠️ 문제 발생:**

* 이번에는 2페이지의 **11번, 12번, 13번** 책들이 **앞쪽에 책이 추가되면서 순번이 뒤로 밀려버립니다.**
* 다시 **같은 `OFFSET 10`, `LIMIT 10`** 으로 2페이지를 요청하면, 시스템은 여전히 **새롭게 11번부터 20번** 책을 가져오려고 합니다.
* 하지만 이 "새로운 2페이지"는 **추가 전 2페이지의 일부 데이터 (원래 11번, 12번, 13번 책)** 를 **다시 보여주거나, 페이지가 밀리면서 데이터가 중복**되어 보이는 현상이 발생할 수 있습니다.

#### 4. 🔑 핵심 요약: 절대 위치 의존의 함정

Offset 방식 페이지네이션의 문제점은 결국 **`OFFSET` 이라는 '절대적인 위치'** 에 지나치게 의존한다는 데 있습니다.

* **절대 위치 의존:**  Offset 방식은 데이터 목록에서 **정해진 순번**을 기준으로 데이터를 가져옵니다.
* **데이터 변경에 민감:** 책(데이터)이 추가되거나 삭제되면 **순번이 바뀌면서**, 이전에 계산된 `OFFSET` 값이 더 이상 **정확한 위치를 가리키지 않게 됩니다.**
* **결과 문제:**  이로 인해 사용자가 기대한 **데이터 구간과 실제 반환되는 데이터 구간 사이에 중복 또는 누락 문제**가 발생할 수 있는 것이죠.

---

## 3. Cursor Pagination (cursor 방식)

### 3.1 기본 개념 및 동작 원리
**Cursor Pagination** 은 "책갈피"(CreepHyp - 栞 노래 좋아요)처럼, 
마지막으로 조회한 항목의 고유 값(예: `id`나 `created_at`)을 기준으로 다음 페이지 데이터를 요청하는 방식입니다.  
마치 책갈피를 사용하여 읽던 페이지를 기억하고, 다음 내용을 이어서 읽는 것과 유사합니다.  
특히 대용량 데이터셋 또는 실시간 데이터 스트리밍 환경에서 뛰어난 성능과 데이터 일관성을 제공합니다. 
이 방식은 데이터가 추가되거나 삭제되어도 일관성을 유지할 수 있어 대용량 데이터 처리에 매우 유리합니다.

### 3.2 백엔드 SQL 예제

#### 예제 6: 기본 SQL 쿼리 (cursor 없이 첫 페이지)
```mysql
-- 첫 페이지는 cursor 없이 LIMIT만 사용
SELECT id, name, created_at
FROM products
ORDER BY id ASC
LIMIT ?;
```
> **설명:**  
> 첫 페이지 요청 시 cursor가 없으므로 단순히 LIMIT만 적용합니다.

#### 예제 7: SQL 쿼리 (cursor 사용하여 다음 페이지 조회)
```mysql
-- 마지막으로 조회한 id (예: 5)를 cursor로 사용하여 다음 5개 데이터를 조회
SELECT id, name, created_at
FROM products
WHERE id > ?
ORDER BY id ASC
LIMIT ?;
```
> **설명:**  
> :last_id 값 이후의 데이터를 가져와, 순차적으로 페이지를 이동합니다.

### 3.3 프론트엔드 JavaScript 예제

#### 예제 8: Graphql + Apollo를 이용한 Cursor 기반 데이터 요청
```javascript

import { useLazyQuery } from '@apollo/client';


// GraphQL Query 정의: cursor 기반 상품 목록 조회
const GET_PRODUCTS_CURSOR = gql`
  query GetProducts($limit: Int!, $cursor: String) {
    products(limit: $limit, after: $cursor) {
      edges {
        node {
          id
          name
          price
          createdAt
          __typename // Apollo Client 캐싱에 중요
        }
        cursor
      }
      pageInfo {
        hasNextPage // 다음 페이지 존재 여부
        endCursor   // 마지막 아이템 cursor
      }
    }
  }
`;

let nextCursor = null; // 다음 페이지 cursor를 저장 (초기값 null)
                       
// cursor 기반 페이지네이션 커스텀 훅
const useProductsCursorPagination = (pageSize) => {

  const [fetchProducts, { data, loading, error }] = useLazyQuery(GET_PRODUCTS_CURSOR); // useLazyQuery 훅 사용: 수동 트리거

  const fetchedProductsData = data?.products?.edges?.map(({ node }) => node) || []; // 상품 데이터 추출
  const hasNextPage = data?.products?.pageInfo?.hasNextPage || false; // 다음 페이지 존재 여부 추출
  nextCursor = data?.products?.pageInfo?.endCursor || null; // 다음 cursor 업데이트: pageInfo의 endCursor 사용

  // 초기 상품 로드 함수 (첫 페이지): cursor 없이
  const loadInitialProducts = async () => {
    try {
      await fetchProducts({ variables: { limit: pageSize, cursor: null } }); // cursor 없이 첫 페이지 요청
    } catch (fetchError) {
      console.error("초기 상품 로드 오류:", fetchError);
    }
  };

  // 다음 페이지 로드 함수
  const loadNextPage = async () => {
    if (hasNextPage && nextCursor) { // 다음 페이지가 있고 cursor가 있을 때만 요청
      try {
        await fetchProducts({ variables: { limit: pageSize, cursor: nextCursor } }); // cursor 사용하여 다음 페이지 요청
      } catch (fetchError) {
        console.error("다음 페이지 상품 로드 오류:", fetchError);
      }
    } else {
      console.log("더 이상 페이지 없음 또는 다음 cursor 없음.");
      // UI 처리: "다음 페이지" 버튼 비활성화 등
    }
  };

  // 훅 반환값: 로딩, 에러, 상품, 페이지 정보, 로드 함수
  return {
    loading,
    error,
    products: fetchedProductsData,
    hasNextPage,
    loadInitialProducts,
    loadNextPage
  };
};
```
```jsx


import React, { useEffect } from 'react';

const ProductListWithCursorPagination = ({ pageSize = 10 }) => { // pageSize prop으로 설정 가능
  const {
    loading,
    error,
    products,
    hasNextPage,
    loadInitialProducts,
    loadNextPage
  } = useProductsCursorPagination(pageSize); 

  useEffect(() => { 
    loadInitialProducts();
  }, []);

  if (loading) return <p>Loading products...</p>
  if (error) return <p>Error : {error.message}</p>

  return (
    <div>
      <h2>Product List (Cursor Pagination with GraphQL & Apollo)</h2>
      <ul>
        {products.map(product => (
          <li key={product.id}>
            {product.name} - Price: ${product.price} - Created At: {product.createdAt}
          </li>
        ))}
      </ul>

      {hasNextPage && ( // 다음 페이지 버튼 렌더링: hasNextPage가 true일 때만
        <button onClick={loadNextPage} disabled={loading}>
          {loading ? 'Loading next page...' : 'Load Next Page'}
        </button>
      )}

      {!hasNextPage && products.length > 0 && <p>No more products.</p>} {/* 마지막 페이지 표시 */}
      {products.length === 0 && !loading && !error && <p>No products available.</p>} {/* 상품 없을 때 표시 */}
    </div>
  );
};

export default ProductListWithCursorPagination;

```


> **설명:**
> - cursor가 있는 경우 URL에 파라미터로 추가하여 API 요청을 보냅니다.
> - 서버 응답에서 `nextCursor` 값을 받아 다음 페이지 요청에 활용합니다.

### 3.4 장단점

#### 👍 장점 

- **🚀 압도적인 성능 (대용량 데이터)**: Offset 기반 Pagination 과 달리, 데이터셋 크기에 관계없이 **일정한 쿼리 성능**을 유지합니다. 특히 대용량 데이터, 무한 스크롤에 최적화 (서비스 확장성 확보)
- **👍 데이터 일관성**: 페이지 이동 중 데이터 삽입/삭제가 발생해도 데이터 중복/누락 문제 발생 가능성이 **매우 낮습니다.** (실시간 데이터 스트리밍, 채팅 등에 적합)
- **✨ 확장성**: 서버는 cursor만 관리하면 되므로, 클라이언트 상태 (페이지 번호 등) 를 관리할 필요가 없어 서버 **확장성이 뛰어납니다.** (MSA 환경에 적합)

#### 👎 단점 

- **🧱 구현 복잡도 증가**: Cursor 생성, 관리, 클라이언트 전달 등 구현 복잡도가 Offset-based Pagination 보다 상대적으로 높습니다. (초기 개발 및 유지보수 난이도 증가)
- **🧭 페이지 번호 탐색 제한**: 페이지 번호 기반 직접 이동이 **불가능하거나 구현이 매우 어렵습니다.** 순차적인 "다음 페이지" 탐색만 가능합니다. (일반적인 웹 페이지 Pagination UI 와는 다름)
- **🔑 cursor 관리 및 보안**: cursor 유효 기간 관리, cursor 정보 노출 방지 등 cursor 관리 및 보안 이슈를 고려해야 합니다. (보안 및 안정성 중요)


### 3.5 ⚠️단점: Cursor Pagination, 왜 페이지 번호 탐색이 어려울까?

>Cursor 값 자체에는 전체 데이터에서 몇 번째 페이지인지, 총 페이지 수가 몇 페이지인지와 같은 정보가 담겨있지 않습니다. 
Cursor는 단순히 "다음 데이터 묶음을 가져오기 위한 티켓" 같은 역할을 할 뿐이에요. 특정 페이지 번호로 직접 이동하는 기능을 구현하려면, 
서버에서 전체 데이터셋을 스캔하여 해당 페이지에 해당하는 Cursor 값을 별도로 계산해야 하는 복잡한 과정이 필요하며, 이는 Cursor Pagination의 장점인 성능 효율성을 크게 떨어뜨립니다

---

## 4. Offset Pagination vs. Cursor Pagination: 언제 어떤 방식을 선택할까?
### 🤔 어떤 Pagination 방식을 선택해야 할까요?
Pagination 방식 선택에 정답은 없습니다. 실제로 UI/UX, 기존 시스템, 프로젝트 제약 등 다양한 이유로 여러분이 자유롭게 선택하기 어려울 수 있습니다.
데이터 규모와 변경 빈도, 요구 사항, 개발 리소스, 사용자 경험 등을 종합적으로 고려해야 합니다.
결국, 여러 제약 조건을 고려하여 상황에 맞는 최적의 방식을 선택하는 것이 중요합니다.

---


# 🎉 마무리

Offset Pagination과 Cursor Pagination의 개념부터 장단점, 구체적인 SQL 및 JavaScript 예제, 그리고 실제 Mock Data까지 모두 살펴보았습니다.  
프론트엔드와 백엔드 개발자 모두에게 유용한 정보가 되었길 바라며,
질문이나 추가적인 의견이 있다면 댓글로 남겨주시고, 도움이 되셨다면 공유 부탁드립니다. 😊
