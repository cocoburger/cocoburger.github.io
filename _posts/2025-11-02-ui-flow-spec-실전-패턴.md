---
title: "UI Flow Specification 시리즈 2편: 실전 패턴(제안)"
date: 2025-11-02 00:05:00 +0900
categories: [Frontend, Software Engineering, Documentation]
tags: [ui-flow, frontend, state-management, ui-state, software-design, specification]
series: ui-flow-spec
pin: false
toc: true
---

## 들어가며

1편에서 UI Flow Specification(UI Flow Spec)을 "Use Case와 구현 사이의 간극을 메우는 프론트엔드 실무 템플릿"으로 제안했습니다. 이번 글은 그 템플릿을 코드와 바로 연결하는 실전 패턴을 정리합니다.

> 용어 주석: 본 글에서 사용하는 'UI Flow Specification'은 업계 표준 용어가 아니라, 필자가 Use Case와 구현 사이의 간극을 메우기 위해 제안하는 실무 템플릿입니다. 유사 산출물로는 UI/UX Spec, Interaction Spec, Statechart, Acceptance Criteria 등이 있습니다.

### 핵심 요약
- UI Flow Spec → 상태 머신/쿼리/구조로 매핑하면 코드와 1:1 연결이 쉬워집니다.
- Storybook/E2E와 연결하면 UI Flow Spec이 곧 테스트/데모 시나리오가 됩니다.
- 내용은 학습/제안 단계이며, 팀 컨텍스트에 맞춰 가볍게 시작하는 것을 권장합니다.

## UI Flow Spec → 코드 변환 패턴

### 패턴 1: 상태 머신(XState)

```typescript
import { createMachine } from 'xstate';

/** UI Flow Specification: 상품 구매 (Based on Use Case #12) */
export const productPurchaseMachine = createMachine({
  id: 'productPurchase',
  initial: 'idle',
  states: {
    idle: { on: { PURCHASE: 'loading' } },
    loading: {
      invoke: {
        src: 'checkStock',
        onDone: { target: 'paymentReady', actions: 'setProduct' },
        onError: { target: 'error', actions: 'setError' }
      }
    },
    paymentReady: {
      on: { SUBMIT_PAYMENT: 'processing', CANCEL: 'idle' }
    },
    processing: {
      invoke: {
        src: 'processPayment',
        onDone: { target: 'success' },
        onError: { target: 'paymentFailed' }
      }
    },
    success: { type: 'final' },
    error: {},
    paymentFailed: { on: { RETRY: 'paymentReady' } }
  }
});
```

- UI Flow Spec의 상태/전이/이벤트를 그대로 머신으로 옮깁니다.
- Spec의 시나리오 번호를 주석으로 남기면 테스트와 상호참조가 편합니다.

### 패턴 2: TanStack Query + 상태 관리

```typescript
// Step 1: 재고 확인
const stockQuery = useQuery({
  queryKey: ['stock', productId],
  queryFn: checkStock,
  enabled: false,
});

// Step 2: 결제 처리
const paymentMutation = useMutation({
  mutationFn: processPayment,
  onSuccess: () => {
    toast.success('결제 완료!');
    router.push('/order-complete');
  },
  onError: (error: any) => {
    if (error.code === 'INSUFFICIENT_STOCK') toast.error('재고 부족');
  }
});

// UI Flow Spec의 Step을 함수로 캡슐화
const handlePurchase = async () => {
  const stock = await stockQuery.refetch();
  if (stock.data?.available) paymentMutation.mutate(paymentData);
};
```

- Spec의 단계(LOADING → PAYMENT_READY → PROCESSING → SUCCESS)를 쿼리/뮤테이션 라이프사이클과 정렬합니다.
- UI 피드백(토스트/라우팅)은 Spec의 "시스템 반응"을 기준으로 구현합니다.

### 패턴 3: Feature-Sliced Design(FSD)로 구조화

```text
src/
  features/
    product-purchase/           # UI Flow Spec = Feature 단위와 1:1
      model/
        use-purchase-flow.ts    # 훅/상태
        purchase-machine.ts     # XState 머신(선택)
      ui/
        PurchaseButton.tsx      # Step 1
        PaymentModal.tsx        # Step 2
        SuccessAnimation.tsx    # Step 3
      api/
        purchase-api.ts
      spec/
        ProductPurchase.uiflow.md
```

- 파일 체계에 Spec을 함께 두면(PR 동시 리뷰, 변경 이력, 동기화)이 쉬워집니다.

## 디자인/테스트 도구와의 연계

### Storybook: 상태별 Story 매핑

```typescript
export default { title: 'Features/ProductPurchase', component: ProductPurchase };
export const Idle = () => <ProductPurchase state="idle" />;
export const Loading = () => <ProductPurchase state="loading" />;
export const PaymentReady = () => <ProductPurchase state="paymentReady" />;
export const Processing = () => <ProductPurchase state="processing" />;
export const Success = () => <ProductPurchase state="success" />;
export const Error = () => <ProductPurchase state="error" />;
```

- UI Flow Spec의 각 상태를 Story로 고정하면 디자이너/QA와 소통이 쉬워집니다.

### Playwright: 시나리오 기반 E2E

```typescript
test('UI Flow Spec: 상품 구매 - 정상 플로우', async ({ page }) => {
  await page.goto('/product/123');                 // Step 1: IDLE
  await page.click('button:has-text("구매하기")'); // Step 2: LOADING
  await expect(page.getByRole('dialog')).toBeVisible(); // PAYMENT_READY
  await page.fill('[name="cardNumber"]', '1234-5678-9012-3456');
  await page.click('button:has-text("결제")');     // PROCESSING → SUCCESS
  await expect(page.getByText('결제가 완료되었습니다')).toBeVisible();
});
```

- 테스트의 Given/When/Then을 Spec의 상태/전이/피드백과 1:1로 맞춥니다.

## 오늘부터 실천하기(체크리스트)
- 현재 진행 중인 복잡 UI 1개 선택
- UI Flow Spec으로 상태/전이/피드백을 1시간 내 작성
- XState 또는 TanStack Query로 최소 기능 구현(2시간)
- Storybook 상태 스토리와 Playwright E2E 1개씩 추가(1시간)

## 한계와 주의
- Use Case는 구현 독립적 문서이며, UI 디테일은 의도적으로 범위 밖입니다.
- UI Flow Spec은 이를 보완하는 실무 템플릿이므로, 팀/제품 맥락에 맞춰 간소화/확장하세요.
- 접근성/성능/보안 등은 별도의 가이드와 함께 검토가 필요합니다.

## 마치며
학습/제안 단계에서 시작하는 작은 실험이 팀 커뮤니케이션과 품질 개선으로 이어질 수 있다고 기대합니다. 피드백과 사례 제보를 환영합니다.

### 다음 글 예고
- 사례 A: 다단계 회원가입 UI Flow Spec
- 사례 B: 실시간 협업(문서 편집) UI Flow Spec
