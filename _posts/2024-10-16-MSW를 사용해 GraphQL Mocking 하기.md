---
title: "MSW와 GraphQL을 활용한 프론트엔드 선행 개발: 경험과 회고"
date: 2024-10-16 11:30:00 +0900
categories: [개발, 프론트엔드]
tags: [msw, graphql, mocking, 회고]
pin: false
toc: true
---

안녕하세요, 제가 최근 프로젝트에서 경험한 Mock Service Worker(MSW)와 GraphQL을 활용한 프론트엔드 선행 개발에 대해 이야기해보려고 합니다.
특히 이 과정에서 겪은 실제 어려움과 깨달은 점들을 솔직히 공유하고자 합니다.

## 목차

1. 프로젝트 배경
2. MSW와 GraphQL 소개 및 선택 이유
3. 개발 과정에서의 겪은 어려움
4. 실제 서버로의 마이그레이션
5. 회고 및 결론

## 1. 프로젝트 배경

새로운 서비스 제공을 위해 개발하게 되었습니다. 프로젝트 초기에 직면한 주요 문제는 다음과 같았습니다:

- 백엔드 API가 아직 개발되지 않은 상태
- 디자인과 데이터 구조가 확정되지 않음
- 여유롭지 않은 개발 일정 (다른 프로젝트로 인해 개발 시작 일정이 늦어졌지만, 개발 마감 기한은 동일한 경우)

## 2. MSW와 GraphQL 소개 및 선택 이유

### MSW (Mock Service Worker) 소개

MSW는 API 모킹(mocking)을 위한 라이브러리로, 서비스 워커를 활용하여 네트워크 요청을 가로채고 모의 응답을 제공합니다.

<h4 style="color: #2c3e50;">주요 특징</h4>
<ul style="list-style-type: none; padding-left: 0;">
  <li style="margin-bottom: 10px;">
    <span style="font-weight: bold; color: #3498db;">✓ 실제 API 시뮬레이션:</span> 실제 네트워크 요청과 유사한 환경을 제공
  </li>
  <li style="margin-bottom: 10px;">
    <span style="font-weight: bold; color: #3498db;">✓ 다양한 환경 지원:</span> 브라우저와 Node.js 환경 모두에서 사용 가능
  </li>
  <li style="margin-bottom: 10px;">
    <span style="font-weight: bold; color: #3498db;">✓ 개발 프로세스 개선:</span> 테스트, 개발, 디버깅 과정에서 유용하게 활용
  </li>
</ul>

### GraphQL 소개

GraphQL은 API를 위한 쿼리 언어이자 런타임으로, 클라이언트가 필요한 데이터를 정확히 요청할 수 있게 해줍니다.

<h4 style="color: #2c3e50;">주요 특징</h4>
<ul style="list-style-type: none; padding-left: 0;">
  <li style="margin-bottom: 10px;">
    <span style="font-weight: bold; color: #3498db;">✓ 유연한 데이터 요청:</span> 클라이언트가 필요한 데이터만 정확히 요청 가능
  </li>
  <li style="margin-bottom: 10px;">
    <span style="font-weight: bold; color: #3498db;">✓ 단일 엔드포인트:</span> 여러 리소스를 하나의 요청으로 가져올 수 있음
  </li>
  <li style="margin-bottom: 10px;">
    <span style="font-weight: bold; color: #3498db;">✓ 강력한 타입 시스템:</span> 데이터의 구조와 타입을 명확히 정의
  </li>
</ul>

### MSW와 GraphQL 선택 이유

1. **선행 개발 가능:** 백엔드 개발과 병렬적으로 프론트엔드 개발 진행 가능
2. **유연한 데이터 구조 설계:** GraphQL의 스키마 정의를 통한 명확한 데이터 구조 설계
3. **실제 API와 유사한 환경 구축:** MSW를 통해 실제 API와 유사한 개발 환경 제공
4. **빠른 프로토타이핑:** 백엔드 의존성 없이 빠른 기능 구현 및 테스트 가능

## 3. 개발 과정에서의 겪은 어려움

### MSW 설정 및 데이터 생성 함수 개발

MSW 설정은 비교적 간단했지만, 목 데이터 생성 함수 개발에는 예상보다 많은 시간이 소요되었습니다.

- **복잡한 데이터 구조**: 실제 애플리케이션의 데이터 구조가 복잡해질수록, 이를 정확히 모방하는 목 데이터를 만드는 것도 복잡해졌습니다.
- **다양한 시나리오 고려**: 다양한 엣지 케이스와 에러 상황을 고려한 mock 데이터를 만들어야 했습니다.
- **GraphQL Query 생성**: GraphQL query를 만들어야 하는 코드량 증가로 인한 작업량이 많았습니다.
- **기획 변경에 대한 유연성 부족**: 기획이 변경될 때마다 데이터 구조를 유동적으로 수정해야 했지만, MSW로 구현한 목업 데이터는 이러한 변경에 빠르게 대응하기 어려웠습니다. 이로 인해 기획 변경 시마다 추가적인 작업 시간이 필요했고, 때로는 기존에 작성한 코드의 대규모 수정이 불가피했습니다.


```typescript
const GET_USER = gql`
  query GetUser($id: ID!) {
    user(id: $id) {
      id
      name
      email
      posts {
        id
        title
        content
      }
      followers {
        id
        name
      }
    }
  }
`;

// 쿼리 사용
const { data: users, error, loading } = useQuery(GET_USER, {
  variables: { first: 5 },
});

// MSW 핸들러
export const handlers = [
  graphql.query('GetUser', (req) => {
    const { first = 5, after } = req.variables
    const start = after ? parseInt(after, 10) + 1 : 0
    const data = generateUserList(start, first)
    return HttpResponse.json({
      data: {
        reserveList: {
          id: data.databaseId,
          name: data.expertName,
          age: data.age,
          pageInfo: {
            hasNextPage: data.pageInfo.hasNextPage,
            endCursor: data.pageInfo.endCursor,
          },
        },
      },
    })
  }),
];
```

이러한 함수를 만들고 유지보수하는 데 예상 외로 많은 시간이 소요되었습니다.

## 4. 실제 서버로의 마이그레이션

서버 개발이 완료된 후, 실제 서버로 마이그레이션하는 과정에서 여러 어려움을 겪었습니다:

- **데이터 구조 불일치**: 초기에 백엔드 팀과 데이터 구조를 공유했음에도 불구하고, 실제 구현된 API의 데이터 구조가 예상과 달랐습니다.
- **피드백 부재**: 개발 과정에서 백엔드 팀으로부터 충분한 피드백을 받지 못해, 많은 부분을 가정하에 개발했습니다.
- **데이터 매핑 작업**: 결과적으로 프론트엔드의 데이터 처리 로직을 거의 처음부터 다시 작성해야 했습니다.



## 5. 회고 및 결론
벡엔드 개발과 독립적으로 진행할 수 있어 빠르게 프로토타입을 제공할 수 있다는 점이 좋았습니다.
MSW을 활용한 프론트엔드 선행 개발 전략은 분명 장점이 있지만, 첫 도입시에는 set-up을 위한 시간이 필요했습니다.

프로젝트마다 마감기한이 정해져있고 팀원별로 개발 진행 속도와 개발 스타일이 다르므로
공동으로 정의하고 관리하는 process가 잘 구축이 안되어 있다면, 프론트엔드에서 너무 세세한 데이터 구조까지 다 만들어서 mocking하는 건 좀 과하다라고 생각합니다. 
그러므로 다음 번에 동일한 상황이 발생한다면
UI에 꼭 필요한 데이터만 간단하게 mocking해서 연결할 것 같습니다.

#### 예시
```typescript
const mockUserData = {
  name: "홍길동",
  age: 30,
  email: "hong@example.com",
  // UI에 필요한 다른 필드들...
};

// 이 Mock 데이터를 MSW로 서빙
graphql.query('GetUserProfile', (req, res, ctx) => {
  return res(ctx.data({ user: mockUserData }));
});
```

