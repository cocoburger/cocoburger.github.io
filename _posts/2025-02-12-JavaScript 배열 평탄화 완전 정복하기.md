---
title: "JavaScript 배열 평탄화 완전 정복: 재귀와 reduce로 flatten 함수 직접 만들기 (feat. Currying 함수로 변환)"
date: 2025-02-12 01:00:00 +0900
categories: [개발, JavaScript]
tags: [javascript, array, flatten, reduce, recursion, currying, functional-programming]
pin: false
toc: true
---

배열은 JavaScript에서 가장 많이 사용되는 자료 구조 중 하나입니다. 때로는 배열 내에 또 다른 배열이 중첩된 복잡한 구조를 다뤄야 할 때가 있습니다. 이러한 **중첩 배열**을 **평탄화(flatten)** 하여 단일 레벨 배열로 만드는 것은 데이터 처리 효율성을 높이고 코드 가독성을 개선하는 데 매우 중요합니다.

JavaScript는 ES2019부터 `Array.prototype.flat()` 이라는 편리한 평탄화 메서드를 제공합니다. 하지만,  **재귀 호출**과 **`reduce` 메서드**를 사용하여 `flatten` 함수를 직접 구현하는 것은 JavaScript의 동작 원리를 깊이 이해하는 데 도움이 됩니다.

본 글에서는 재귀와 `reduce`를 활용하여 중첩 배열을 평탄화하는 `flatten` 함수를 만들고, 예제와 함께 동작 방식을 상세히 설명합니다. 심화 과정으로 Currying을 적용하여 함수형 프로그래밍 기법까지 확장해 보겠습니다.

## 1. `flatten` 함수란? - 중첩 배열

`flatten` 함수는 **중첩된 배열 구조를 특정 깊이까지 펼쳐서 단일 레벨의 배열로 변환하는 함수**입니다. 마치 층층이 쌓인 상자를 열어 내용물을 꺼내 펼치는 것과 같습니다.

**핵심 기능:**

* **중첩 구조 제거**: 배열 속의 배열을 한 단계 위로 "펼쳐"줍니다.
* **깊이 제어**: 평탄화 깊이를 설정하여 원하는 레벨까지만 펼칠 수 있습니다. (기본 깊이: 1, `Infinity` 설정 시 완전 평탄화)

**`flatten` 함수 활용 예시:**

### 1.1. API 응답 데이터 가공

API에서 JSON 형태로 중첩된 데이터를 받을 때, `flatten`을 사용하면 데이터를 1차원 배열로 쉽게 가공하여 분석 및 활용도를 높일 수 있습니다.

```javascript
const nestedApiResponse = {
  items: [
    { id: 1, name: '제품 A', options: [[{ color: 'red' }, { size: 'M' }], [{ style: 'classic' }]] }, // options가 2차원 배열
    { id: 2, name: '제품 B', options: [[{ color: 'blue' }]] },                                   // options가 2차원 배열
  ],
};

// API 응답 데이터를 제품 옵션만 추출하여 1차원 배열로 평탄화
const productOptions = flatten(nestedApiResponse.items.map(item => item.options), 2); // depth를 2로 설정 (혹은 Infinity)
console.log(productOptions);
// 출력: [{ color: 'red' }, { size: 'M' }, { style: 'classic' }, { color: 'blue' }]
```

### 1.2. 카테고리 및 하위 카테고리 검색

쇼핑몰 카테고리 데이터와 같이 계층 구조로 표현된 데이터를 검색해야 할 때, `flatten`으로 모든 카테고리를 평탄화하면 검색 로직을 단순화하고 효율성을 높일 수 있습니다.

```javascript
const complexApiResponse = {
  categories: [
    {
      name: '의류',
      subCategories: [
        {
          name: '상의',
          items: [
            { id: 1, name: '티셔츠', options: [{ size: 'M' }] },
            { id: 2, name: '셔츠', options: [{ color: 'white' }] },
          ],
        },
        {
          name: '하의',
          items: [
            { id: 3, name: '바지', options: [{ size: 'L' }] },
          ],
        },
      ],
    },
    {
      name: '잡화',
      subCategories: [
        {
          name: '신발',
          items: [
            { id: 4, name: '운동화', options: [{ color: 'navy' }] },
          ],
        },
      ],
    },
  ],
};

// 모든 카테고리, 서브 카테고리, 제품의 옵션을 1차원 배열로 평탄화 (예시: 제품 옵션만 추출)
const allProductOptions = flatten(
  complexApiResponse.categories.map(category =>
    category.subCategories.map(subCategory =>
      subCategory.items.map(item => item.options)
    )
  ),
  Infinity // 완전 평탄화 (생략 가능)
);

console.log(allProductOptions);
// 출력: [[{ size: 'M' }], [{ color: 'white' }], [{ size: 'L' }], [{ color: 'navy' }]]  -> 아직 중첩 (원래 의도는 내부 객체들을 빼내는 것일 수도)

// 만약 객체 자체를 평탄화하고 싶다면 (하지만 데이터 구조가 깨질 수 있음)
const allOptionObjects = flatten(allProductOptions, Infinity);
console.log(allOptionObjects);
// 출력: [{ size: 'M' }, { color: 'white' }, { size: 'L' }, { color: 'navy' }]  -> 객체만 남음 (원하는 형태에 따라 다를 수 있음)


// 카테고리와 제품 이름을 함께 평탄화 (더 현실적인 예시)
const allCategoriesAndProducts = flatten(
  complexApiResponse.categories.map(category => [
    category.name,
    flatten(category.subCategories.map(subCategory => [
      subCategory.name,
      flatten(subCategory.items.map(item => [item.name, item.options]), 2) // 제품 이름과 옵션
    ]), 2) // 서브 카테고리
  ]), 2 // 카테고리
);

console.log(allCategoriesAndProducts);
// 출력 예시 (원하는 형태에 따라 조정): ['의류', '상의', '티셔츠', [{ size: 'M' }], '셔츠', [{ color: 'white' }], '하의', '바지', [{ size: 'L' }], '잡화', '신발', '운동화', [{ color: 'navy' }]]
```

## 2. `flatten` 함수 구현 (재귀 & `reduce` 활용)

이제 재귀 호출과 `reduce` 메서드를 사용하여 `flatten` 함수를 직접 구현해 보겠습니다.

```javascript
const flatten = (array, depth = Infinity) => {
  return array.reduce((acc, item) => {
    return depth > 0 && Array.isArray(item)
      ? acc.concat(flatten(item, depth - 1)) // 재귀 호출: depth 감소 후 하위 배열 평탄화
      : acc.concat(item);                    // 배열이 아니거나 depth=0이면 그대로 누적
  }, []); // 초기 누적값은 빈 배열
}
```

**코드 상세 분석:**

* **`flatten(array, depth = Infinity)`**: 함수는 평탄화할 배열 `array`와 깊이 `depth` (기본값 Infinity)를 인수로 받습니다.
* **`array.reduce((acc, item) => { ... }, [])`**: `reduce` 메서드로 배열 순회 및 평탄화된 요소 누적을 수행합니다.
  * `acc` (accumulator): 누적되는 평탄화된 배열. 초기값은 빈 배열 `[]`.
  * `item` (current value): 현재 순회 중인 배열 요소.
* **`depth > 0 && Array.isArray(item)`**: 평탄화 깊이가 남아있고, 현재 요소가 배열인지 확인하는 조건식입니다.
  * **조건 만족 시**: `acc.concat(flatten(item, depth - 1))`:
    * 요소 `item`이 배열이므로 **재귀적으로 `flatten` 함수를 호출**합니다.
    * `depth`를 1 감소시켜 재귀 호출하여 하위 배열을 평탄화합니다.
    * 재귀 호출 결과를 현재 누적 배열 `acc`에 `concat`하여 병합합니다.
  * **조건 불만족 시**: `acc.concat(item)`:
    * 요소 `item`이 배열이 아니거나, `depth`가 0이면 `item`을 그대로 누적 배열 `acc`에 `concat`합니다.
* **`[]`**: `reduce` 메서드의 초기 누적값으로 빈 배열을 설정합니다.

## 3. `flatten` 함수 동작 과정

**입력 값:**

```javascript
const arr = [1, [2, [3, [4]]]];
```

**함수 호출:**

```javascript
const result = flatten(arr, Infinity);
```

**동작 과정:**

flatten 함수는 reduce 메서드를 사용해서 배열을 순회합니다. reduce는 마치 우리가 빈 바구니(acc, 누적값) 를 들고 배열의 각 요소(item)를 하나씩 확인하면서, 요소에 따라 바구니에 무언가를 담거나, 더 깊숙이 들어가서 작업을 하는 것과 비슷합니다.

### **최상위 호출**: `flatten([1, [2, [3, [4]]]], Infinity)`

* `depth` (평탄화 깊이)는 `Infinity` 이므로, 최대한 깊이까지 평탄화를 진행합니다.
* `reduce` 메서드가 시작되고, **초기 빈 바구니 (`acc = []`)** 를 준비합니다.

  * **첫 번째 아이템 (`item = 1`) 처리**:
    * `Array.isArray(1)` 은 `false` 입니다. (숫자 1은 배열이 아니죠!)
    * 따라서, 숫자 `1` 을 **그대로 바구니(`acc`)에 넣습니다**: `acc.concat(1)` -> **현재 바구니 상태: `[1]`**

  * **두 번째 아이템 (`item = [2, [3, [4]] ]`) 처리**:
    * `Array.isArray([2, [3, [4]]])` 은 `true` 입니다. (배열입니다!)
    * `depth` 도 아직 `Infinity` 로 남아있으므로, **더 깊숙이 들어가서** 하위 배열 `[2, [3, [4]]]` 에 대해 **새로운 `flatten` 함수를 재귀적으로 호출**합니다.

  ### **➡️ 재귀 호출 1단계**: `flatten([2, [3, [4]]], Infinity)`

  * **새로운 `flatten` 함수가 호출**되었고, **새로운 빈 바구니 (`acc = []`)** 를 준비합니다. (이 바구니는 **재귀 호출 1단계**에서 사용할 바구니입니다!)

    * **첫 번째 아이템 (`item = 2`) 처리**:
      * `Array.isArray(2)` 는 `false` 입니다.
      * 숫자 `2` 를 **현재 단계의 바구니(`acc`)에 넣습니다**: `acc.concat(2)` -> **현재 바구니 상태: `[2]`**

    * **두 번째 아이템 (`item = [3, [4]]`) 처리**:
      * `Array.isArray([3, [4]])` 은 `true` 입니다. (배열입니다!)
      * `depth` 는 여전히 `Infinity` 이므로, **또 다시 더 깊숙이 들어가서** 하위 배열 `[3, [4]]` 에 대해 **`flatten` 함수를 재귀 호출**합니다.

    ### **➡️➡️ 재귀 호출 2단계**: `flatten([3, [4]], Infinity)`

    * **또 다시 새로운 `flatten` 함수가 호출**! **새로운 빈 바구니 (`acc = []`)** 준비! (이번 바구니는 **재귀 호출 2단계** 용!)

      * **첫 번째 아이템 (`item = 3`) 처리**:
        * `Array.isArray(3)` 는 `false` 입니다.
        * 숫자 `3` 을 **현재 단계의 바구니(`acc`)에 넣습니다**: `acc.concat(3)` -> **현재 바구니 상태: `[3]`**

      * **두 번째 아이템 (`item = [4]`) 처리**:
        * `Array.isArray([4])` 은 `true` 입니다. (배열입니다!)
        * `depth` 는 아직 `Infinity` 이므로, **한 번 더 깊숙이!** 하위 배열 `[4]` 에 대해 **`flatten` 함수 재귀 호출**!

      ### **➡️➡️➡️ 재귀 호출 3단계**: `flatten([4], Infinity)`

      * **마지막 재귀 호출!** **새로운 빈 바구니 (`acc = []`)** 준비 완료! (드디어 **재귀 호출 3단계**!)

        * **유일한 아이템 (`item = 4`) 처리**:
          * `Array.isArray(4)` 는 `false` 입니다.
          * 숫자 `4` 를 **현재 단계의 바구니(`acc`)에 넣습니다**: `acc.concat(4)` -> **현재 바구니 상태: `[4]`**

        * **더 이상 처리할 요소가 없습니다**.  **재귀 호출 3단계가 종료**되고, 현재 바구니 상태인 **`[4]` 를 "결과"로서 반환**합니다.

    ### **⬅️⬅️ 재귀 호출 2단계로 돌아오기**:

    * **재귀 호출 3단계가 `[4]` 라는 결과를 반환**했습니다!
    * 이제 **재귀 호출 2단계**에서 진행하던 `reduce` 메서드가 이어서 진행됩니다.
    * **재귀 호출 3단계의 결과 `[4]`** 를 **현재 단계의 바구니(`acc = [3]`)에 합칩니다**: `[3].concat([4])` -> **현재 바구니 상태: `[3, 4]`**

    * **더 이상 처리할 요소가 없습니다**.  **재귀 호출 2단계가 종료**되고, 현재 바구니 상태인 **`[3, 4]` 를 "결과"로서 반환**합니다.

  ### **⬅️ 재귀 호출 1단계로 돌아오기**:

  * **재귀 호출 2단계가 `[3, 4]` 라는 결과를 반환**했습니다!
  * 이제 **재귀 호출 1단계**에서 진행하던 `reduce` 메서드가 다시 진행됩니다.
  * **재귀 호출 2단계의 결과 `[3, 4]`** 를 **현재 단계의 바구니(`acc = [2]`)에 합칩니다**: `[2].concat([3, 4])` -> **현재 바구니 상태: `[2, 3, 4]`**

  * **더 이상 처리할 요소가 없습니다**.  **재귀 호출 1단계가 종료**되고, 현재 바구니 상태인 **`[2, 3, 4]` 를 "결과"로서 반환**합니다.

### **최상위 호출로 돌아오기**:

* **재귀 호출 1단계가 `[2, 3, 4]` 라는 결과를 반환**했습니다!
* 이제 **최상위 호출**에서 진행하던 `reduce` 메서드가 마지막으로 진행됩니다.
* **재귀 호출 1단계의 결과 `[2, 3, 4]`** 를 **최상위 호출의 바구니(`acc = [1]`)에 합칩니다**: `[1].concat([2, 3, 4])` -> **최종 바구니 상태: `[1, 2, 3, 4]`**

* **최상위 호출의 `reduce` 메서드도 모든 요소 순회를 마쳤습니다**.  **최종 바구니 상태 `[1, 2, 3, 4]` 가 `flatten` 함수의 최종 결과로서 반환**됩니다!

**최종 결과:**

```javascript
[1, 2, 3, 4];
```

**요약**:

`flatten` 함수는 **재귀 호출**을 통해 배열 속의 배열을 계속 파고 들어가서, 가장 안쪽의 요소부터 순서대로 꺼내어 **하나의 바구니(`acc`)에 차곡차곡 담는 방식**으로 동작합니다.  각 재귀 호출은 자신의 바구니를 가지고 작업을 하고, 작업이 끝나면 자신의 바구니를 "결과"로서 반환하여 상위 호출의 바구니에 합쳐지는 방식으로 최종 결과가 만들어집니다.

이제 `flatten` 함수의 동작 방식이 좀 더 명확하게 이해되셨기를 바랍니다!

### 3.2. 예제 2: 제한된 깊이 평탄화 (depth = 2)

**입력 배열:**

```javascript
const arr = [1, [2, [3, [4]]]];
```

**함수 호출:**

```javascript
const result = flatten(arr, 2);
```

**동작 과정:**

* **최상위 호출**: `flatten([1, [2, [3, [4]]]], 2)` - `depth = 2 > 0`, `acc = []`
  * `item = 1`: `Array.isArray(1) === false` -> `acc.concat(1)` -> `[1]`
  * `item = [2, [3, [4]]]`: `Array.isArray([2, [3, [4]]]) === true` -> **재귀 호출**: `flatten([2, [3, [4]]], 2 - 1)` -> `flatten([2, [3, [4]]], 1)`

* **재귀 호출 1단계**: `flatten([2, [3, [4]]], 1)` - `depth = 1 > 0`, `acc = []`
  * `item = 2`: `Array.isArray(2) === false` -> `acc.concat(2)` -> `[2]`
  * `item = [3, [4]]`: `Array.isArray([3, [4]]) === true` -> **재귀 호출**: `flatten([3, [4]], 1 - 1)` -> `flatten([3, [4]], 0)`

* **재귀 호출 2단계**: `flatten([3, [4]], 0)` - `depth = 0` -> **재귀 멈춤, `array.slice()` 반환**: `[3, [4]]` (얕은 복사)

* **재귀 호출 종료 및 결과 병합**:
  * 재귀 2단계 -> `[3, [4]]` 반환, 재귀 1단계 `acc = [2]` 에 병합 -> `[2, 3, [4]]` 반환
  * 재귀 1단계 -> `[2, 3, [4]]` 반환, 최상위 호출 `acc = [1]` 에 병합 -> `[1, 2, 3, [4]]` 반환

**최종 결과:**

```javascript
[1, 2, 3, [4]];
```

**설명:** `depth = 2`로 설정했으므로, 최상위 배열과 1단계 중첩 배열까지만 평탄화되었습니다. `[3, [4]]` 부분은 `depth` 제한으로 인해 그대로 유지됩니다.

## 4. 심화 과정: Currying으로 `flatten` 함수 변환하기 (Currying을 복습하려고 추가했습니다 ^.^)

### 4.1. Currying 이란? - 함수를 더욱 유연하게! 

**Currying**은 여러 인수를 받는 함수를 단일 인수를 받는 함수의 체인으로 변환하는 함수형 프로그래밍 기법입니다. **부분 적용(partial application)** 을 통해 함수 재사용성과 유연성을 높입니다.

**핵심 아이디어:**

* **일반 함수**: `f(a, b, c)`
* **Currying 함수**: `curried_f(a)(b)(c)`

**Currying 장점:**

* **함수 재사용성**: 일부 인수만 미리 적용하고, 나머지는 나중에 제공하여 함수를 재활용 용이.
* **코드 가독성**: 함수 역할 분리 및 모듈화로 코드 이해도 향상.
* **함수 합성**: 함수를 조합하여 복잡한 기능 구현에 용이.

**Currying 예시 (곱셈 함수):**

```javascript
// 일반 함수
function multiply(a, b) {
  return a * b;
}

// Currying 함수
function curriedMultiply(a) {
  return function(b) {
    return a * b;
  };
}

const multiplyBy5 = curriedMultiply(5); // 5를 미리 적용
console.log(multiplyBy5(3)); // 15 (3을 나중에 적용)
console.log(multiplyBy5(10)); // 50 (10을 나중에 적용)
console.log(curriedMultiply(5)(3)); // 15 (한 번에 호출 가능)
```

### 4.2. `flatten` 함수 Currying 변환

`flatten` 함수를 Currying 해보겠습니다. 깊이(`depth`)를 먼저 받고, 나중에 배열(`array`)을 받는 `curryFlatten` 함수를 만들어 보겠습니다.

```javascript
function curryFlatten(depth) {
  return function(array) {
    return array.reduce((acc, item) => {
      return depth > 0 && Array.isArray(item)
        ? acc.concat(curryFlatten(depth - 1)(item)) // 재귀 호출 시 커링 함수 재사용
        : acc.concat(item);
    }, []);
  };
}
```

**`curryFlatten` 함수 분석:**

* **`curryFlatten(depth)` (외부 함수)**: `depth`를 먼저 받아 클로저에 저장하고, 내부 함수를 반환합니다.
* **`function(array) { ... }` (내부 함수)**: 배열 `array`를 나중에 받아 실제 평탄화 로직을 수행합니다. 재귀 호출 시 `curryFlatten(depth - 1)(item)` 처럼 커링된 함수를 재사용합니다.

**`curryFlatten` 함수 사용 예시:**

```javascript
const arr = [1, [2, [3, [4]]]];

const flattenDepth2 = curryFlatten(2); // 깊이 2로 설정된 flatten 함수 생성
const resultDepth2 = flattenDepth2(arr);
console.log('Depth 2 평탄화:', resultDepth2);
// 출력: Depth 2 평탄화: [ 1, 2, 3, [ 4 ] ]

const flattenDeeply = curryFlatten(Infinity); // 완전 평탄화 함수 생성
const resultDeeply = flattenDeeply(arr);
console.log('완전 평탄화:', resultDeeply);
// 출력: 완전 평탄화: [ 1, 2, 3, 4 ]

const anotherArray = [5, [6, [7]]];
const resultDepth2_another = flattenDepth2(anotherArray); // flattenDepth2 재사용
console.log('다른 배열 Depth 2 평탄화:', resultDepth2_another);
// 출력: 다른 배열 Depth 2 평탄화: [ 5, 6, [ 7 ] ]
```

### 4.3. 화살표 함수로 더 간결하게

ES6 화살표 함수를 사용하면 `curryFlatten` 함수를 더 간결하게 표현할 수 있습니다.

```javascript
const curryFlatten = depth => array =>
  array.reduce((acc, item) =>
    depth > 0 && Array.isArray(item)
      ? acc.concat(curryFlatten(depth - 1)(item))
      : acc.concat(item), []);
```

## 5. 결론 - JavaScript 배열 평탄화

본 글에서는 재귀 호출과 `reduce` 메서드를 사용하여 중첩 배열을 평탄화하는 `flatten` 함수를 직접 구현하고, 깊이 있는 예제와 함께 동작 방식을 자세히 알아보았습니다. 더 나아가 Currying 기법을 적용하여 함수를 더욱 유연하고 재사용 가능하게 만드는 방법까지 학습했습니다.

이제 JavaScript 배열 평탄화에 대한 깊은 이해와 함수형 프로그래밍 역량까지 갖추게 되었습니다.  `flatten` 함수와 Currying을 자유자재로 활용하여 더욱 효율적이고 가독성 높은 JavaScript 코드를 작성해 보세요!
