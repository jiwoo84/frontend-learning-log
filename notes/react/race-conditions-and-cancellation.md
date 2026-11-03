# React 비동기 경쟁 상태와 요청 취소 (AbortController)

최초 기록일: 2026-11-03 (Asia/Seoul)  
상태: 비동기 검색 요청의 경쟁 상태, 스코프 격리, AbortController와 AbortError 분기 처리 당일 학습 마무리. 설명 후 자기 정리 및 핵심 이유 설명 확인.

---

## 1. 출발한 문제: 비동기 검색과 경쟁 상태 (Race Condition)

검색창에 사용자가 `'r'`을 입력한 후 곧바로 `'react'`를 입력하는 상황을 가정한다.

```jsx
function SearchComponent({ query }) {
  const [results, setResults] = useState([]);
  const [isLoading, setIsLoading] = useState(false);

  useEffect(() => {
    setIsLoading(true);

    fetchResults(query)
      .then((data) => {
        setResults(data);
      })
      .finally(() => {
        setIsLoading(false);
      });
  }, [query]);

  // ...
}
```

### 발생하는 문제
1. **요청 역전**: 네트워크 지연으로 인해 뒤늦게 보낸 `'react'` 요청이 먼저 응답(200 OK)하고, 먼저 보낸 `'r'` 요청이 나중에 도착할 수 있다.
2. **상태 불일치**:
   - `'react'` 응답 도착 시: `results`가 `'react'` 결과로 갱신되고, `finally`에 의해 `isLoading`이 `false`가 됨. (아직 `'r'` 요청은 백그라운드 네트워크에서 실행 중임에도 로딩이 조기 종료됨)
   - `'r'` 응답 도착 시: 뒤늦게 도착한 `'r'` 응답이 `results`를 덮어씀.
   - **최종 상태**: 검색창에는 `'react'`가 적혀 있는데, 화면에는 `'r'`의 결과가 표시되고 로딩은 종료되어 있는 심각한 UI/상태 불일치(Race Condition)가 발생한다.

---

## 2. 해결책 1: Effect 정리 함수와 `ignore` 플래그

React 공식 문서에서는 Effect 정리(cleanup) 함수와 렉시컬 스코프 플래그를 활용해 이전 응답을 무시하는 패턴을 안내한다.

```jsx
useEffect(() => {
  let ignore = false;
  setIsLoading(true);

  fetchResults(query)
    .then((data) => {
      if (!ignore) {
        setResults(data);
      }
    })
    .finally(() => {
      if (!ignore) {
        setIsLoading(false);
      }
    });

  return () => {
    ignore = true;
  };
}, [query]);
```

### 동작 원리: 렉시컬 스코프와 클로저
- **호출마다 독립된 스코프**: `query`가 변경되어 Effect가 재실행될 때마다 Effect 함수가 새로 호출되므로, `let ignore = false;`는 매번 새로운 메모리 공간에 독립적으로 선언된다.
- **클로저에 의한 격리**:
  - 1회차('r') Effect의 `then/finally` 콜백과 정리 함수는 1회차의 `ignore₁`을 캡처한다.
  - 2회차('react') Effect의 콜백과 정리 함수는 2회차의 `ignore₂`를 캡처한다.
- **정리 함수 실행 타이밍**:
  - React는 2회차 Effect를 실행하기 **직전**에 1회차 정리 함수를 먼저 호출한다.
  - 따라서 1회차의 `ignore₁`만 `true`로 바뀌며, 2회차의 `ignore₂`는 여전히 `false`로 유지된다.
  - `'r'` 응답이 뒤늦게 오더라도 `ignore₁ === true`이므로 상태 업데이트가 안전하게 무시된다.

### `ignore` 플래그의 한계
- UI의 상태 불일치는 해결하지만, **실제 네트워크 요청 자체는 브라우저와 서버 사이에서 취소되지 않고 끝까지 진행**된다.
- **서버 리소스 낭비**: 백엔드와 데이터베이스는 이미 버려질 이전 검색어에 대해 무거운 쿼리와 정렬 연산을 끝까지 수행한다.
- **브라우저 네트워크 병목 (Connection Starvation)**: 브라우저의 도메인당 동시 연결 수(HTTP/1.1 기준 보통 6개)를 불필요한 이전 요청들이 점유하여, 정작 최신 요청이나 다른 중요 리소스의 다운로드가 큐에 밀려 지연된다.

---

## 3. 해결책 2: 브라우저 표준 `AbortController`를 통한 네트워크 취소

브라우저 표준 API인 `AbortController`를 사용하면 이전 요청 자체를 네트워크 레벨에서 물리적으로 중단(abort)할 수 있다.

### 올바른 구현 코드

```jsx
useEffect(() => {
  const controller = new AbortController();
  setIsLoading(true);

  async function search() {
    try {
      const res = await fetchResults(query, { signal: controller.signal });
      const data = await res.json();
      setResults(data);
      setIsLoading(false);
    } catch (error) {
      // 1. 의도된 취소(AbortError)는 정상 흐름이므로 조용히 무시
      if (error.name === "AbortError") {
        return;
      }
      // 2. 실제 서버 장애나 네트워크 단절인 경우에만 사용자 알림 및 로딩 종료
      showErrorToast("검색 중 오류가 발생했습니다.");
      setIsLoading(false);
    }
  }

  search();

  return () => {
    controller.abort();
  };
}, [query]);
```

### 핵심 주의점 및 실무 판단 기준

#### 1) `AbortError` 분기 처리의 필요성
- `controller.abort()`가 호출되면 브라우저는 즉시 전송을 끊고, `fetch`는 **Promise를 거부(reject)하며 `name: 'AbortError'`인 `DOMException`을 던진다.**
- JavaScript 비동기 흐름상 **취소 또한 에러의 일종으로 `catch` 블록에 전달**된다.
- 만약 `catch` 블록에서 이를 분기 처리하지 않고 일반 오류처럼 다루면:
  - 사용자가 타이핑할 때마다 이전 요청들이 취소되면서 화면에 "데이터 조회 실패" 에러 팝업/토스트가 연달아 뜨는 치명적인 UI 버그가 발생한다.
  - Sentry 등 사내 에러 모니터링 도구에 수많은 허위 장애 알람이 집계된다.
- 따라서 `if (error.name === 'AbortError') return;` 처럼 의도된 취소임을 확인하고 아무런 에러 처리 없이 조용히 종료해야 한다.

#### 2) 로딩 상태 조기 종료 방지
- `finally` 블록에서 무조건 `setIsLoading(false)`를 호출하면, 취소된 이전 요청이 `AbortError`를 거쳐 `finally`를 실행하면서 **새로 진행 중인 최신 요청의 로딩 스피너를 도중에 꺼버리는 버그**가 생긴다.
- 따라서 로딩 종료는 성공 시점(`try` 끝 / `then`)과 실제 실패 시점(`catch`의 진짜 에러)에만 명시적으로 꺼주어야 한다.

#### 3) `useEffect`와 `async` 함수
- `useEffect(async () => { ... }, [])` 처럼 콜백 자체에 `async`를 붙이면 함수가 `Promise`를 반환하게 된다.
- React는 Effect 콜백의 반환값으로 오직 **정리(cleanup) 함수**만을 기대하므로, `async` 함수를 직접 넘기면 정리 함수가 정상 작동하지 않는다.
- 따라서 Effect 내부에 별도의 `async` 함수를 선언해 호출하거나, Promise `.then().catch()` 체인을 사용해야 한다.

#### 4) 보충: `try...catch`와 `async/await`, `Promise`의 관계
- `try...catch`는 원래 동기(Synchronous) 코드용 에러 처리 문법이다.
- 과거 `Promise` 시절에는 비동기 에러가 다른 실행 시점(콜 스택)에서 발생했기 때문에 전통적인 `try...catch`로 잡지 못하고 `.catch()` 메서드를 사용해야 했다.
- `async/await`는 Promise 위에서 동작하는 문법적 설탕(Syntactic Sugar)으로, `await`가 실패한 Promise(`reject`)를 만나면 동기식 에러를 `throw`하도록 변환한다. 덕분에 비동기 코드도 익숙한 `try...catch`로 감싸 일관되게 에러를 처리할 수 있게 되었다.
- `await`는 일반 함수 내부에서는 쓸 수 없으며, 반드시 `async` 함수 내부(또는 최상위 모듈 레벨)에서만 유효하다.

---

## 4. 최종 요약

- **경쟁 상태 원인**: 네트워크 지연 시간은 가변적이므로 요청 순서와 응답 도착 순서는 일치하지 않는다.
- **`ignore` 플래그**: Effect 정리 함수와 클로저 스코프를 이용해 이전 응답 반영을 무시할 수 있으나, 네트워크 및 서버 자원 낭비는 남는다.
- **`AbortController`**: 불필요한 이전 요청을 네트워크 레벨에서 즉시 중단하여 자원을 절약한다. 단, 취소 시 던져지는 `AbortError`를 반드시 분기하여 사용자 에러 노출 및 로딩 조기 종료를 방지해야 한다.

---

## 5. 출처
- [React: Synchronizing with Effects (Fetching data)](https://react.dev/learn/synchronizing-with-effects#fetching-data)
- [MDN: AbortController](https://developer.mozilla.org/en-US/docs/Web/API/AbortController)
- [MDN: Fetch API - Canceling a request](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API#canceling_a_request)
