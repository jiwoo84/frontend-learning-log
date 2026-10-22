# TypeScript 타입 추론·유니온·좁히기

학습·기록일: 2026-10-22 (Asia/Seoul)
상태: 당일 학습 마무리. 타입 추론·유니온 의미·실행 전 검사 역할을 자기 설명했고, return 이후 숫자만 남는 이유를 설명했다. 마지막 좁히기의 목적 설명은 피드백으로 보완했으며 보완 후 자기 설명·독립 적용·지연 기억은 미확인이다.

## 1. JavaScript의 동적 타입과 TypeScript의 타입 추론

JavaScript에서는 변수의 타입이 고정되지 않는다. 값에 타입이 있으며, 같은 변수에 숫자를 넣었다가 문자열을 넣을 수 있다.

```js
let price = 1000;
console.log(typeof price); // 예상: "number"
price = "무료";
console.log(typeof price); // 예상: "string"
```

typeof는 현재 값의 타입을 확인한다. 이를 TypeScript의 정적 타입 추론과 혼동하지 않는다. TypeScript는 실행 전에 코드를 검사하며, 다음 선언에서는 초기값을 근거로 price의 타입을 number로 추론한다.

```ts
let price = 1000;
price = "무료"; // 예상 타입 오류: string을 number에 대입
```

타입을 직접 쓰지 않아도 검사할 수 있다는 뜻이다. 보충 설명: 모든 타입 추론이 초기값만으로 이뤄지는 것은 아니다. 이번에는 초기값이 있는 let 변수 선언을 다뤘다. const의 리터럴 타입이나 문맥에 의한 추론은 아직 학습하지 않았다.

학습자의 최초 답변은 number 추론과 문자열 대입 오류를 정확히 예측했지만 이를 JavaScript의 동적 타입 때문이라고 설명했다. 이후 JavaScript 값의 타입과 TypeScript의 실행 전 추론을 구분해 설명했다.

## 2. 유니온 타입: 여러 타입 중 하나

유니온 타입(union type)은 여러 타입 중 하나가 될 수 있음을 나타내며 |로 연결한다.

```ts
let price: number | string = 1000;
price = "무료"; // 허용
```

number | string은 숫자 또는 문자열이라는 뜻이다. 하나의 값이 동시에 두 타입이라는 뜻은 아니다. 타입 표시는 가능한 값의 범위를 나타내며 숫자를 문자열로 바꾸거나 그 반대로 바꾸지 않는다.

```ts
function formatPrice(price: number | string) {
  return price.toUpperCase(); // 예상 타입 오류
}
```

함수는 숫자도 받을 수 있다. toUpperCase는 문자열 메서드이므로 이 위치에서 무조건 호출할 수 없다. 실제 숫자에 호출하면 올바르지 않은 결과만 생기는 것이 아니라 TypeError가 발생한다. 학습자는 숫자일 수 있으므로 호출이 허용되지 않는다는 이유를 설명했고, 구체적인 런타임 오류 종류는 에이전트가 보완했다.

## 3. 타입 좁히기: 코드 위치별로 가능한 타입 줄이기

타입 좁히기(narrowing)는 조건과 실행 흐름을 근거로 특정 위치에서 가능한 타입의 범위를 더 구체적으로 판단하는 과정이다. 값을 변환하는 작업이 아니며, 좁히기 자체가 실행 중 값을 차단하는 별도 기능도 아니다.

```ts
function formatPrice(price: number | string) {
  if (typeof price === "string") {
    return price.toUpperCase();
  }
  return price.toFixed(0);
}
```

함수 입구에서는 number 또는 string이다. typeof 조건이 참인 분기에서는 string으로 판단하므로 toUpperCase를 쓸 수 있다. 문자열인 경우 return으로 함수가 끝나므로 마지막 줄까지 도달하는 값은 number다. 이 위치에서는 숫자 메서드 toFixed를 쓸 수 있다. 예상 반환값은 formatPrice("free")가 "FREE", formatPrice(1000)가 "1000"이다. toFixed의 반환값도 문자열이라는 점은 보충 설명이다.

조건문은 실행 중 실제로 분기를 선택하고, TypeScript는 실행 전에 그 조건을 근거로 타입을 판단한다. 둘의 역할을 구분한다. 이 예제는 문자열 입력을 거부하지 않는다. 문자열과 숫자를 각각 안전하게 처리한다.

학습자는 문자열이면 if에서 걸러지므로 아래에서는 숫자라고 판단할 수 있다고 설명했다. 에이전트는 단순히 if를 거쳤기 때문이 아니라 return으로 문자열 경로가 끝난다는 점을 명확히 했다.

```ts
function formatPrice(price: number | string) {
  if (typeof price === "string") {
    console.log(price.toUpperCase());
  }
  return price.toFixed(0); // 예상 타입 오류
}
```

이 변형에서는 문자열도 로그를 출력한 뒤 마지막 줄에 도달한다. 마지막 줄에서는 여전히 number | string이므로 toFixed를 무조건 호출할 수 없다. 이 변형은 에이전트의 설명이며 학습자의 독립 풀이로 기록하지 않는다.

## 4. 실행 전 검사와 실행 중 동작

```ts
function double(value: number) {
  return value * 2;
}

double(3);   // 허용, 예상 결과 6
double("3"); // 예상 타입 오류
```

학습자가 지적했듯이 편집기와 컴파일러의 타입 검사에서 잘못된 인자를 미리 잡는 것이 TypeScript의 핵심 장점이다. 타입 선언이 실행 중 변환을 하지 않는다는 설명이 이 장점을 부정하는 것은 아니다.

일반적인 JavaScript 출력에서는 매개변수 타입 표시가 제거된다.

```js
function double(value) {
  return value * 2;
}
```

타입 검사를 우회했거나 검증되지 않은 외부 값이 들어온 경우를 정상적인 TypeScript 직접 호출과 구분해야 한다. JavaScript 실행 중 double("3")은 * 연산자의 변환 규칙 때문에 6이 되고, double("무료")는 NaN이 된다. TypeScript의 number 선언이 변환한 것이 아니다. 외부 데이터의 실제 형태를 확인해야 한다면 실행 중 검사 코드를 작성해야 한다.

보충 설명: 타입 오류 진단과 JavaScript 출력 중단은 별개이며 컴파일러 설정에 따라 달라진다. 모든 오류가 자동으로 실행 불가능해지는 것은 아니다. 이 설정 자체는 이번 학습에서 평가하지 않았다.

## 5. 실제 답변과 피드백

아래는 원문 인용이 아닌 기술적 의미의 요약이다.

- 초기값으로 number가 추론되고 문자열 대입이 오류라는 최초 판단은 정확했다. 동적 타입과 정적 추론을 연결한 이유는 설명으로 교정했다.
- 유니온 매개변수에 toUpperCase를 바로 호출할 수 없는 이유로 숫자 가능성을 들었다.
- 문자열이면 먼저 반환하므로 아래에는 숫자가 남는 흐름을 설명했다.
- 타입 선언이 자동으로 값을 변환하지 않는다고 답했다.
- 마지막 정리 중 유니온·좁히기의 용어 설명이 충분하지 않다고 지적했다. 에이전트가 정의·문법·예제를 다시 연결했다. 이를 학습자의 기억 실패로 단정하지 않는다. 앞으로 새 용어는 의미와 예제를 충분히 연결한 뒤 자기 설명을 요청한다.
- number 인자에 문자열을 넘기는 오류를 실행 전에 잡아내는 것이 TypeScript의 목적임을 학습자가 정확히 강조했다. 에이전트의 런타임 한계 설명이 검사 장점을 가린 점을 바로잡았다.
- 최종 답변: number | string은 숫자 또는 문자열이며, 타입 검사는 값을 변경하지 않으므로 특정 타입일 때만 통과시키는 좁히기가 필요하다고 설명했다.
- 최종 피드백: 유니온 의미와 타입 선언의 비변환은 맞다. 좁히기의 직접적인 필요는 여러 가능성 중 해당 위치의 타입을 알아야 타입별 연산을 안전하게 사용할 수 있다는 데 있다. 조건문이 분기를 수행하고 TypeScript가 그 흐름에서 타입을 좁힌다. 입력을 반드시 거부하는 것은 아니며 실행 전 검사도 오류 예방에 도움이 된다.

자료 없이 정리하도록 안내했지만 실제 참고 여부는 미확인이다. 용어 재설명 이후의 답변이며 마지막 보완을 학습자가 독립적으로 설명했다고 기록하지 않는다. 당일 학습은 여기서 마치고 정식 복습 점수·반복 오류 횟수는 추가하지 않는다.

## 6. 검증 범위와 다음 확인

예제는 설명을 위한 예상 결과이며 실제 컴파일·실행 검증하지 않았다. 독립적인 코드 수정, null 처리, unknown과 외부 데이터 검증, 판별 유니온은 아직 확인하지 않았다. 다음 복습에서는 return 유무에 따른 타입과 안전한 메서드 사용을 배운 범위 안에서 확인한다. 첫 복습 후보일은 2026-10-23이다.

## 출처

- [TypeScript Handbook — Everyday Types](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html): 타입 추론, 유니온과 공통 연산.
- [TypeScript Handbook — Narrowing](https://www.typescriptlang.org/docs/handbook/2/narrowing.html): typeof 검사와 제어 흐름에 따른 좁히기.
- [TypeScript Handbook — The Basics](https://www.typescriptlang.org/docs/handbook/2/basic-types.html): 실행 전 검사, 타입 제거와 출력 설정.
