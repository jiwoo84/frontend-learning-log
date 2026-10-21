# 스코프와 클로저

학습·기록: 2026-10-21 (Asia/Seoul)
상태: 당일 학습 마무리. 설명 후 자기 정리와 종합 예제 적용 근거가 있으나 실제 자료 참고 여부와 지연 기억 유지는 미확인이다.

## 1. 변수 이름을 찾는 범위

스코프는 변수 이름을 사용할 수 있는 범위다. 함수 안에서 이름을 찾으면 현재 스코프부터 시작해 코드에 중첩된 바깥 스코프를 차례로 확인한다. 처음 찾은 변수를 사용하며, 바깥에서 안쪽 스코프로 들어가 검색하지 않는다. 같은 이름이어도 다른 스코프의 변수는 별개일 수 있다.

```js
const message = "바깥";
function showMessage() {
  const message = "안쪽";
  console.log(message);
}
showMessage();
console.log(message);
```

예상 출력은 안쪽 → 바깥이다. 안쪽 선언은 그 범위에서 바깥의 같은 이름을 가리지만 바깥 변수의 값을 바꾸지 않는다. 학습자는 이 출력과 검색 방향을 처음부터 정확히 설명했다.

let과 const는 블록 스코프다. 함수 본문뿐 아니라 if 등의 블록 안에 선언해도 범위가 생긴다. 이번에는 함수 중첩을 중심으로 다뤘으며 블록·var 차이의 독립 적용은 확인하지 않았다.

## 2. 호출 위치가 아니라 정의 위치: 렉시컬 스코프

```js
const message = "바깥";
function showMessage() {
  console.log(message);
}
function run() {
  const message = "run 안쪽";
  showMessage();
}
run();
```

예상 출력은 바깥이다. showMessage는 전역에 정의되어 있으므로 자신의 스코프 → 전역 순으로 찾는다. 자신을 호출한 run의 지역 변수는 검색 경로에 들어가지 않는다. 이것이 정의된 코드 위치로 바깥 스코프가 정해지는 렉시컬 스코프다. 함수 호출의 흐름과 변수 검색 경로를 구분해야 한다.

학습자는 처음에는 run 안쪽이라고 예측했다. 검색 방향 자체는 맞았지만 호출한 함수의 스코프를 바깥 스코프로 생각한 부분을 설명으로 교정했다.

## 3. 바깥 함수가 끝난 뒤에도 접근하는 클로저

```js
function run() {
  const message = "run 안쪽";
  function showMessage() {
    console.log(message);
  }
  return showMessage;
}
const show = run();
show();
```

예상 출력은 run 안쪽이다. run()은 내부 함수를 반환하고 종료한다. show는 반환된 함수를 가리키며, 나중에 호출해도 그 함수는 만들어질 때 연결된 환경의 message에 접근할 수 있다.

클로저는 함수와 그 함수가 만들어질 때의 렉시컬 환경이 결합된 것이다. 흔히 바깥 변수에 접근하는 함수를 클로저라고 부른다. 함수 반환은 이 성질을 보여주는 예이며 반환되어야만 클로저가 되는 것은 아니다. 함수 실행이 끝나도 이후 함수가 사용하는 바깥 변수에 대한 접근이 유지된다. 당시 값 전체를 복사해 고정하는 기능은 아니다.

학습자는 정의 위치가 run 내부이므로 해당 문자열을 출력한다고 정확히 설명했다.

## 4. 같은 호출의 변수는 공유하고, 다른 호출은 분리된다

```js
function createCounter() {
  let count = 0;
  function increase() {
    count += 1;
    console.log(count);
  }
  return increase;
}
const counterA = createCounter();
const counterB = createCounter();
counterA(); // 예상: 1
counterA(); // 예상: 2
counterB(); // 예상: 1
```

createCounter를 실행할 때마다 별도의 환경과 count가 만들어진다. A를 두 번 호출하면 첫 환경의 같은 count가 0 → 1 → 2가 되고, B는 두 번째 환경의 count를 0 → 1로 바꾼다. counterA()는 반환된 increase만 실행하므로 let count = 0을 다시 실행하지 않는다. 학습자는 연속 호출의 1, 2와 별도 카운터의 1을 맞혔으며 다른 변수를 가진다고 설명했다.

한 호출에서 여러 함수를 반환하면 같은 지역 변수를 공유할 수 있다.

```js
function createCounter() {
  let count = 0;
  return {
    increase() { count += 1; },
    getCount() { return count; },
  };
}
const counter = createCounter();
counter.increase();
counter.increase();
console.log(counter.getCount()); // 예상: 2
```

객체의 increase()와 getCount()는 함수를 정의하는 메서드 문법이다. 두 함수가 같은 실행 환경에서 만들어졌으므로 하나가 바꾼 count를 다른 하나가 읽는다. 바깥 코드가 지역 변수 count를 직접 읽거나 대입할 수는 없지만 제공된 함수를 통해 조작할 수 있다. 독립적인 상태와 허용된 동작을 묶는 사용 사례다. 이 사용 상황은 에이전트가 설명했으며 학습자가 별도 구현한 것은 아니다.

## 5. 변수 접근과 이미 계산한 결과의 차이

```js
function createCounter() {
  let count = 0;
  const message = `현재 값: ${count}`;
  return {
    increase() { count += 1; },
    print() {
      console.log(count);
      console.log(message);
    },
  };
}
const counter = createCounter();
counter.increase();
counter.print();
```

예상 출력은 1과 현재 값: 0이다. 템플릿 문자열은 해당 표현식이 실행될 때 count를 읽어 문자열을 만든다. message에는 자동으로 다시 계산되는 식이 아니라 완성된 문자열이 저장된다.

1. createCounter 실행 중 count를 0으로 초기화한다.
2. message 초기화에서 count를 읽어 현재 값: 0 문자열을 만든다.
3. increase가 같은 count를 1로 바꾼다. message를 바꾸는 코드는 없다.
4. print가 두 변수를 읽으면 변경된 count와 변경되지 않은 message를 얻는다.

현재 값이 반영된 문장이 필요하면 print 안에서 console.log(`현재 값: ${count}`)처럼 호출 시점에 계산한다. 클로저는 바깥 변수 접근을 유지하며 이미 계산한 결과를 자동으로 갱신하지 않는다.

학습자는 처음에는 1과 현재 값: 1로 예측했다. 설명 후 message는 선언문 실행 시 count를 사용해 결정되고 count가 달라져도 자동 업데이트되지 않는다고 자기 말로 정리했다. 이는 설명 후 교정 근거다.

## 6. 함수 자체와 호출 결과 저장

```js
const read = counter.getCount;        // 함수 자체
const saved = counter.getCount();     // 지금 호출한 결과인 숫자
counter.increase();
console.log(read());                  // 호출할 때 count 읽기
console.log(saved);                   // 앞서 저장한 숫자
```

위 코드는 4절의 getCount를 가진 counter를 사용하는 설명용 발췌다. read를 어떤 이름으로, 어디서 호출해도 원래 함수의 렉시컬 환경은 유지된다. saved에는 함수가 아니라 호출 결과가 저장된다.

학습 중 createAccount의 deposit과 getBalance를 반환한 뒤 getBalance를 다른 변수에 담아 호출하는 예제도 설명했다. 메서드를 꺼내도 balance와의 연결이 유지되는 이유는 함수 정의 위치가 바뀌지 않기 때문이다. 보충 설명: 이번 함수들은 this를 사용하지 않는다. this를 사용하는 다른 메서드까지 동일한 호출 결과를 보장한다고 일반화하지 않는다. this는 이번 평가 범위가 아니다.

## 7. 종합 적용 문제와 실제 답변

사용자가 더 자세한 마지막 문제를 요청했다. 정식 복습 3문제 세트가 아니라 당일 학습의 종합 문제 하나다. 설명·실행 결과를 보기 전에 예측하도록 안내했으며 실제 자료 참고 여부는 확인하지 않았다.

```js
function createCounter(start) {
  let count = start;
  const initialMessage = `시작 값: ${count}`;
  return {
    increase() { count += 1; },
    getCount() { return count; },
    print() {
      console.log(initialMessage);
      console.log(`현재 값: ${count}`);
    },
  };
}
const counterA = createCounter(0);
const counterB = createCounter(10);
const readA = counterA.getCount;
counterA.increase();
counterA.increase();
counterB.increase();
function check() {
  const count = 999;
  console.log(readA()); // ①
}
check();
counterA.print(); // ②, ③
counterB.print(); // ④, ⑤
```

아래는 사용자 답변 요약이며 직접 인용이 아니다.

- 최초 출력: ① 0, ② 시작 값: 0, ③ 현재 값: 2, ④ 시작 값: 10, ⑤ 현재 값: 11.
- check의 999는 readA의 스코프에 해당하지 않아 영향이 없다고 설명했다. 에이전트는 정확한 기준이 readA 변수 선언 위치가 아닌 원래 getCount 함수의 정의 위치라고 보완했다.
- A 안의 세 함수는 같은 count를 사용하고 B와는 공유하지 않는다고 설명했다. createCounter 호출마다 다른 환경이 만들어진다는 이유도 정확했다.
- initialMessage는 초기화할 때, 현재 값 문장은 print 실행 중 console.log에 도달할 때 계산된다고 설명했다.

피드백: ①만 2로 교정했으며 나머지는 정확했다. 에이전트가 함수 저장과 호출 결과 저장의 차이를 추가로 설명했지만, 학습자는 마지막에 증가 호출 두 번을 놓쳐 0이라고 했고 이해했다고 밝혔다. 따라서 이 오류를 함수/값 혼동이나 클로저 개념 오해로 확정하지 않는다. 사용자 설명에 따른 실행 순서 누락으로 기록한다. 교정 후 재시험은 하지 않았다.

답변 이후 에이전트가 Node.js v20.20.2에서 이 종합 예제를 실행해 다음 출력을 확인했다.

```text
2
시작 값: 0
현재 값: 2
시작 값: 10
현재 값: 11
```

다른 예제는 예상 결과이며 개별 실행 검증하지 않았다.

## 8. 자기 정리와 다음 확인

종합 문제 이전 자기 정리 요약: 함수는 자신의 스코프부터 정의된 환경의 바깥 스코프를 찾으며, 그 환경에 연결되어 있고 당시 값을 복사해 두는 것은 아니다. 이를 클로저로 설명했다. 표현상 함수와 환경의 결합이라는 점을 보완했다. 사용자가 학습이 부족하다고 느껴 같은 환경을 공유하는 함수, 상태 관리, 미리 계산한 문자열을 추가로 다뤘다.

당일 학습을 마쳤으며 장기 숙달로 확정하지 않는다. 설명 후 자기 설명, 변형 문제의 공유/분리·계산 시점 적용, 마지막 출력 누락과 사용자 해명을 각각 보존한다. 정식 복습 점수나 반복 오류 횟수는 추가하지 않는다.

- 첫 복습 예정: 2026-10-22. 정의 위치, 같은 환경/별도 환경, 변수 읽기와 문자열 계산 시점을 배운 범위에서 확인한다.
- 다음 학습: React state/ref를 렌더별 환경과 연결해 재확인한다.
- 미확인: React 렌더별 클로저 적용, 비동기 콜백·반복문에서의 클로저, 실제 기능 독립 구현, 지연 기억 유지. 이를 모두 오늘 평가하지 않는다.

## 출처

- [MDN: Closures](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Closures) — 렉시컬 스코프, 함수와 환경의 결합, 공유 및 독립 환경. 예제와 학습 기록은 이번 대화를 바탕으로 작성했다.
