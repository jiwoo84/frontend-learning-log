# async/await와 오류 처리

학습 시작·마무리: 2026-10-21 (Asia/Seoul)
상태: 설명 후 이해 및 마지막 변형 예제의 실행 순서·이유 설명 확인. 실제 자료 참고 여부와 장기 기억 유지는 미확인.

## 1. async 함수도 호출하면 바로 실행된다

`async`는 함수 전체를 마이크로태스크 큐에 넣는 표시가 아니다. 함수를 호출하면 일반 함수처럼 바로 콜 스택에서 실행하고, 첫 `await`에 도달할 때까지 동기적으로 진행한다.

```js
async function run() {
  console.log("A");
  await Promise.resolve("결과");
  console.log("B");
}
run();
console.log("C");
```

예상 출력은 A → C → B다. run 호출로 A를 출력하고, 이미 성공한 Promise를 만든 다음 await에서 함수 실행을 잠시 중단한다. 호출한 쪽은 계속 진행해 C를 출력한다. 현재 동기 코드가 끝난 뒤 마이크로태스크로 run을 재개해 B를 출력한다. 이벤트 루프가 run 전체를 큐에서 꺼내 처음 실행하는 것이 아니다.

학습자의 최초 답변은 C → A → B였다. async 함수 전체가 큐에 들어간다고 생각했으나, 직접 호출과 await 이후 재개를 구분해 설명한 뒤 후속 예제에서는 올바른 순서를 답했다.

## 2. await는 멈춘 함수의 실행을 이어준다

await는 함수가 아니라 연산자다. 이 학습에서는 async 함수 안에서 Promise를 기다리는 사용법을 다룬다. 현재 함수의 실행을 중단하며 프로그램 전체를 막지 않는다. Promise 결과가 정해지면 실행을 재개할 작업이 마이크로태스크로 예약된다. 이미 성공·실패가 정해져 있어도 await 이후를 즉시 동기적으로 실행하지는 않는다.

“뒤 코드를 통째로 큐에 옮긴다”는 표현보다 “중단한 함수를 재개할 작업을 예약한다”가 정확하다. 재개할 때 함수를 처음부터 다시 호출하지 않는다.

```js
async function run() {
  console.log("A");
  await Promise.resolve("결과");
  console.log("B");
  console.log("D");
}
run();
console.log("C");
```

예상 출력: A → C → B → D. 재개한 뒤 B와 D는 한 흐름에서 동기적으로 실행된다. 각 줄마다 별도 마이크로태스크가 생기지 않는다. 뒤에서 또 await를 만나면 다시 중단한다. 아직 대기 중인 Promise라면 결과가 정해질 때까지 재개하지 않는다. 이 두 조건은 설명했으며 별도 적용 문제로 확인하지 않았다.

## 3. 성공하면 결과값을 받는다

```js
async function run() {
  const value = await Promise.resolve("결과");
  console.log(value);
}
run();
console.log("끝");
```

예상 출력: 끝 → 결과. Promise.resolve는 실행할 콜백이 없어도 주어진 값으로 성공한 Promise를 만드는 동작을 한다. then이 없어서 아무 동작도 하지 않는 것은 아니다. 여기서는 await가 성공 결과를 받아 value를 초기화한다. await 때문에 함수가 중단된 동안 바깥의 끝이 먼저 출력된다.

학습자는 이 예제에서 끝 → 결과와 중단·재개 흐름을 정확히 설명했다. 코드가 큐로 이동한다는 표현은 재개 작업이 예약된다는 뜻으로 보완했다.

## 4. 실패하면 await 자리에서 실패 이유를 던진다

```js
async function run() {
  try {
    const value = await Promise.reject(new Error("요청 실패"));
    console.log("성공:", value);
  } catch (error) {
    console.log("실패:", error.message);
  }
}
run();
console.log("끝");
```

예상 출력: 끝 → 실패: 요청 실패.

1. run이 바로 실행되어 try 안으로 들어간다.
2. new Error가 오류 객체를 만든다. 객체 생성 자체는 오류를 던지는 동작이 아니다.
3. Promise.reject가 이 객체를 실패 이유로 가진 Promise를 만든다. 이 표현식은 await로 중단하기 전에 평가된다.
4. await에서 함수 실행을 중단하고, 바깥 코드가 끝을 출력한다.
5. 재개할 때 await가 실패 이유인 Error 객체를 던진다.
6. value 초기화와 성공 출력은 건너뛰고 catch로 이동한다. catch의 error는 Error 객체이며 error.message가 메시지 문자열이다.

Promise.reject가 직접 catch를 호출하는 것이 아니다. await가 실패 이유를 던지고 try/catch가 이를 처리한다. await 자체가 실패를 복구하거나 처리까지 마치는 것도 아니다.

학습자는 출력 순서를 맞혔으나 처음에는 reject/new Error가 직접 catch로 전달한다고 설명했다. 생성, 실패한 Promise 생성, await의 던지기, catch의 처리 단계를 구분해 보완했다.

## 5. 실패 이유는 Error 객체에 한정되지 않는다

| 표현식 | catch가 받는 값 |
| --- | --- |
| await Promise.reject() | undefined |
| await Promise.reject("hi") | 문자열 hi |
| await Promise.reject(new Error("hi")) | Error 객체 |

```js
async function run() {
  try {
    const value = await Promise.reject("hi");
    console.log("성공");
  } catch (error) {
    console.log(error);         // hi
    console.log(error.message); // undefined
  }
}
```

위 함수는 호출하면 hi와 undefined를 출력할 것으로 예상한다. 문자열에는 message 속성이 없기 때문이다. 반면 reject()로 인자를 생략하면 error가 undefined이므로 error.message 접근 자체에서 새로운 TypeError가 발생한다. 실패한 await는 정상 결과를 반환하지 않으므로 value에 hi나 undefined가 할당된다는 뜻이 아니다.

사용자는 인자가 없거나 평문 문자열인 reject도 catch로 넘어가는지 질문했다. 어떤 값이든 실패 이유가 될 수 있으며 await가 그 값을 그대로 던진다고 설명했다. 보충 설명: 일반적으로 Error 객체를 사용하면 메시지와 디버깅 정보를 함께 다루기 좋다. 이 권장 방식이나 undefined의 속성 접근 오류는 별도 독립 적용으로 확인하지 않았다.

## 6. 마지막 점검과 실제 답변

설명과 노트를 보지 않고 답하도록 안내하고 다음 코드를 제시했다.

```js
async function run() {
  console.log("A");
  try {
    const value = await Promise.reject("hi");
    console.log("B", value);
  } catch (error) {
    console.log("C", error);
  }
  console.log("D");
}
run();
console.log("E");
```

학습자의 실제 답변 요약(직접 인용 아님): async 함수도 즉시 실행하므로 A를 출력한다. 실패한 Promise를 await하면서 멈추고 바깥 코드가 E를 출력한다. 재개하면 hi를 던져 catch에서 C와 hi를 출력하고 이후 D를 출력한다.

평가: A → E → C hi → D와 핵심 이유가 정확하다. catch 뒤의 일반 코드까지 이어지는 변형 조건을 설명했다. 답변의 “Chi” 표기는 C와 hi를 함께 출력한다는 의미로 보며 오개념으로 세지 않는다. value 초기화가 완료되지 않는다는 점은 명시적으로 설명하지 않았으므로 에이전트가 보충했다. “바깥 함수 E”는 함수 호출부 뒤의 console.log를 가리키는 표현으로 보완했다.

마지막 문제에는 추가 힌트를 주지 않았다. 실제 자료 참고 여부는 미확인이므로 완전한 독립 인출로 단정하지 않는다. 당일 학습을 마무리하며 정식 지연 복습 점수는 부여하지 않는다.

답변 후 에이전트가 Node.js v20.20.2에서 마지막 코드를 실행해 A, E, C hi, D를 확인했다. 브라우저에서는 실행하지 않았다. 이 문서의 다른 예제는 설명용 예상 결과이며 실행 검증했다고 주장하지 않는다.

## 다음 확인과 학습 범위

- 첫 복습 예정일: 2026-10-22. 요청된 복습에서 최초 호출과 재개, 성공값과 실패 이유를 확인한다.
- 다음 새 주제: 스코프와 클로저. 이후 기존 React state/ref 내용과 연결한다.
- 미확인 범위: 여러 await, 대기 중인 Promise, async 함수 반환값, 함수 밖으로 전파되는 실패, await 없는 Promise의 try/catch 처리, 실제 요청 실패 처리.
- 오늘 확인한 예제를 일반적인 async/await 전체 숙달이나 장기 기억으로 확대하지 않는다.

## 출처

- [MDN: await](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/await)
- [MDN: async function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function)
- [MDN: Promise.reject](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/reject)
