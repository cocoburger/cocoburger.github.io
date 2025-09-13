---
title: "Next.js와 CloudFront 문제 해결 가이드"
date: 2025-09-13 22:30:00 +0900
categories: [Next.js, CloudFront, Cache]
tags: [nextjs, cloudfront, cache, rsc, app-router, r2, upload]
pin: false
toc: true
---

Next.js App Router와 CloudFront를 함께 사용하면 강력한 웹 애플리케이션을 구축할 수 있지만, 설정이 조금만 비켜가도 예상치 못한 문제가 발생할 수 있습니다. 이 글에서는 제가 실제로 겪은 세 가지 이슈와 해결 과정을 공유합니다.

- RSC(React Server Components) 페이로드가 CloudFront에 의해 잘못 캐시되어 HTML에 바이트 코드가 노출된 사례
- 클라이언트 이미지 업로드를 서버 라우트로 요청했을 때 CloudFront가 `POST /api/upload/example` 요청을 403으로 차단한 사례
 - Next.js `Image` 최적화 응답이 과도하게 캐시되어 이미지 변경 반영이 지연된 사례

## 사례 1. RSC 페이로드가 HTML로 노출

### 문제 상황

Next.js App Router로 빌드된 애플리케이션을 CloudFront 뒤에 배포한 후, 간헐적으로 페이지가 깨지는 현상을 발견했습니다. 일반적인 HTML이 아닌, `0:[["children"...`과 같은 형태의 알 수 없는 바이트 코드가 화면에 그대로 렌더링되었습니다.

개발자 도구의 네트워크 탭을 확인해보니, 브라우저는 `text/html`을 기대했지만 서버에서 `Content-Type: text/x-component`으로 응답한 것을 확인할 수 있었습니다. 이 문제는 Next.js App Router의 핵심 기능인 RSC와 관련이 깊습니다.

### 원인 분석

Next.js App Router는 서버에 두 가지 유형의 응답을 요청합니다(그리고 별도로 정적 JavaScript 번들을 다운로드합니다).

1.  **HTML 요청**: 사용자가 처음 페이지에 진입할 때 발생하는 일반적인 문서 요청입니다. `Accept` 헤더는 `text/html`입니다.
2.  **RSC 요청**: 클라이언트 사이드 네비게이션 또는 프리패치 시 서버에서 React Server Component를 가져오기 위한 요청입니다. 이때 `Accept` 헤더는 `text/x-component`이며, `_rsc` 쿼리 또는 `RSC: 1` 같은 헤더가 포함될 수 있습니다.

CDN 캐시 관점에서는 HTML과 RSC를 확실히 분리하면 충분합니다. 즉, `_rsc` 쿼리 파라미터나 `RSC` 헤더(그리고 `Accept: text/x-component`)를 캐시 키에 포함해 HTML 캐시와 혼재되지 않도록 해야 합니다.

문제의 원인은 CloudFront가 이 두 종류의 요청을 구분하지 못하고 동일한 캐시 키로 처리했기 때문입니다. CloudFront의 기본 캐시 정책은 `Accept` 헤더나 `_rsc` 쿼리 파라미터를 캐시 키에 포함하지 않습니다. 그 결과, RSC 요청에 대한 응답(`text/x-component`)이 HTML 요청의 캐시로 저장되어 버렸고, 이후 사용자가 동일한 URL에 접속했을 때 캐시된 RSC 페이로드를 그대로 받게 된 것입니다.

### 해결 전략

이 문제를 해결하기 위한 두 가지 방법이 있습니다.

#### 1. 캐시 키 분리 (권장)

가장 이상적인 해결책은 CloudFront가 HTML 요청과 RSC 요청을 별개의 요청으로 인지하도록 캐시 정책을 수정하는 것입니다. 다음과 같이 CloudFront의 캐시 정책에서 캐시 키에 포함될 헤더와 쿼리 문자열을 명시적으로 지정합니다.

-   **Query Strings**: `_rsc`를 포함하도록 설정합니다.
-   **Headers**: `RSC` 헤더를 포함하도록 설정합니다.

이렇게 하면 동일한 URL이라도 `_rsc` 쿼리 파라미터나 `RSC` 헤더의 유무에 따라 다른 캐시 키가 생성되므로, 캐시 오염을 원천적으로 방지할 수 있습니다.

#### 2. RSC 요청 캐시 우회

또 다른 방법은 RSC 요청을 아예 캐시하지 않고 항상 오리진(Next.js 서버)으로 전달하는 것입니다. CloudFront Functions나 Lambda@Edge를 사용하여 들어오는 요청의 헤더를 검사하고, RSC 요청의 특징을 만족하면 캐시를 우회하도록 설정할 수 있습니다.

다음은 RSC 요청을 식별하는 CloudFront Function의 예시 코드입니다.

```javascript
function handler(event) {
  const req = event.request;
  const headers = req.headers || {};
  const accept = (headers['accept'] && headers['accept'].value) || '';
  const rsc = headers['rsc']?.value;
  const hasRscQuery = (req.querystring && Object.prototype.hasOwnProperty.call(req.querystring, '_rsc'));

  // RSC 요청이면 캐시 우회
  if (accept.includes('text/x-component') || rsc === '1' || hasRscQuery) {
    req.headers['cache-control'] = { value: 'no-store' };
  }

  return req;
}
```

이 방법은 구현이 간단하고 확실하지만, RSC 요청에 대해 CDN 캐시의 이점을 누릴 수 없다는 단점이 있습니다.

### 인프라팀과의 협업 및 Terraform을 이용한 해결

Frontend 인프라는 Terraform으로 관리되고 있었기 때문에, CloudFront 설정을 직접 수정하는 대신 인프라 담당자에게 문제를 설명하고 필요한 변경 사항을 요청했습니다.

CloudFront의 캐시 정책에 `RSC` 헤더와 `_rsc` 쿼리 파라미터를 캐시 키로 추가해 달라고 요청했습니다. 인프라팀은 해당 요청을 확인해 Terraform 코드를 수정해 CloudFront 배포에 반영했고, 문제를 해결했습니다.

## 사례 2. CloudFront가 R2 이미지 업로드 POST를 403으로 차단

### 문제 상황

클라이언트 컴포넌트에서 이미지를 업로드하면, 실제 업로드 처리는 Next.js 서버의 `route.ts` API(`POST /api/upload/example`)가 수행하도록 설계했습니다. 그러나 배포 후 업로드 요청이 간헐적으로 403을 반환했고, 서버 로그에는 요청이 도달한 흔적이 없었습니다.

### 단서

응답 헤더(Response Headers)에 있는 다음 두 줄이 그 증거입니다.

- server: CloudFront
- x-cache: Error from cloudfront

이는 사용자의 파일 업로드 요청(POST /api/upload/example)이 우리가 분석했던 Next.js 서버의 `route.ts` 코드에 도달하기도 전에 CloudFront 단에서 차단되고 있다는 의미입니다.

### 원인 분석

우리 배포의 기본 Behavior가 `Allowed HTTP methods`를 `GET, HEAD, OPTIONS`까지만 허용하고 있었고, `/api/*`에 대한 별도 Behavior가 없었습니다. 그 결과 `POST` 업로드 요청이 CloudFront에서 바로 거부(403)되었습니다. 만약 AWS WAF를 붙여두었다면 WAF 차단(역시 403)일 수도 있지만, 이 경우엔 단순히 메서드 허용 범위 미설정이 원인이었습니다.

### 해결

인프라팀에 상황과 원인을 설명했고, CloudFront 배포의 `/api/*` Behavior에서 업로드 API가 `POST`를 포함한 필요한 메서드를 허용하고 API 응답 캐시를 비활성화하도록 요청했습니다. 변경이 반영된 이후 403은 재발하지 않았습니다. (CORS 프리플라이트 `OPTIONS` 허용도 함께 확인했습니다.)

### 배운 점

- 응답 헤더만으로도 문제 지점(CloudFront vs Origin)을 빠르게 특정할 수 있습니다.
- API 경로는 초기에 전용 Behavior를 만들어 허용 메서드·포워딩·캐싱 정책을 명시해두는 것이 안전합니다.

## 사례 3. Next.js Image 컴포넌트와 캐시 문제

### 문제 상황

이미지 교체나 수정 이후에도 사용자 화면에 이전 이미지가 계속 노출되는 현상이 발생했습니다. 특히 `/_next/image` 경유로 최적화된 이미지가 CloudFront에 오래 남아 있어 반영이 지연되었습니다.

### 원인 분석

Next.js `Image` 최적화는 요청 파라미터를 바탕으로 이미지를 가공해 `/_next/image`로 응답하며, 응답에는 `Cache-Control`이 포함됩니다. CloudFront는 이 헤더를 기준으로 결과를 캐시합니다. TTL이 길거나 캐시 정책이 이를 그대로 존중하면 변경 사항 반영이 늦어집니다.

### 해결

`next.config.js`에서 `images.minimumCacheTTL` 값을 조정해 응답의 `Cache-Control: max-age`를 낮췄습니다. 이렇게 하면 CloudFront 캐시 지속 시간이 짧아져 이미지 변경이 더 빨리 반영됩니다.

```javascript
// next.config.js
module.exports = {
  images: {
    minimumCacheTTL: 60, // 기본값은 60초입니다.
  },
};
```

`minimumCacheTTL`은 Next.js가 생성하는 이미지 응답에 포함되는 `Cache-Control`의 `max-age` 값을 결정합니다. 프로젝트 특성에 맞춰 TTL을 조절해 캐시 효율과 신선도 사이의 균형을 맞추는 것이 중요합니다.

### 배운 점

- `/_next/image`는 생성된 결과물이라도 CDN 캐시 전략의 영향을 크게 받습니다.
- 이미지 잦은 변경이 예상되면 TTL을 낮추거나 경로 버전닝/쿼리 해시 등과 함께 운용하는 것이 안전합니다.

## 올바른 캐시 설계의 중요성

사용하는 프레임워크에 대해서 깊이 있는 지식이 얼마나 중요한지 다시 한번 깨닫게 되었습니다. 
특히 Next.js App Router와 같이 복잡한 요청 흐름을 가진 프레임워크를 사용할 때는 CDN의 동작 방식을 정확히 이해하고, 그에 맞는 캐시 전략을 수립해야 합니다.

효과적인 캐시 설계를 위해서는 다음 사항들을 고려해야 합니다.

-   **정적 콘텐츠와 동적 콘텐츠 분리**: 변경이 잦지 않은 정적 에셋(CSS, JS, 이미지 등)은 캐시 TTL을 길게 설정하고, 사용자 데이터와 같이 동적으로 변경되는 콘텐츠는 캐시하지 않거나 TTL을 짧게 설정합니다.
-   **Vary 헤더의 이해**: `Vary` 헤더를 사용하여 동일한 URL에 대해 다른 버전의 콘텐츠가 존재할 수 있음을 CDN에 알려줄 수 있습니다. (예: `Vary: Accept-Encoding`, `Vary: Accept`)
-   **캐시 무효화 전략**: 콘텐츠가 업데이트되었을 때 기존 캐시를 어떻게 무효화할 것인지에 대한 계획이 필요합니다.

## 결론

세 가지 사례 모두 CloudFront가 트래픽을 어떻게 분류·허용·캐시하는지에 대한 이해 부족에서 비롯됐습니다. 첫째는 요청 종류(HTML vs RSC)를 캐시 키로 분리하지 않은 문제, 둘째는 API 업로드 경로의 Behavior에서 허용 메서드·캐싱을 올바르게 설정하지 않은 문제, 셋째는 `/_next/image` 응답 TTL이 사용 시나리오에 비해 길었던 문제였습니다. CloudFront의 Behavior/정책을 정확히 설계하고, 응답 헤더를 통해 문제 지점을 신속히 특정하는 습관을 들이면 재발을 효과적으로 방지할 수 있습니다.
