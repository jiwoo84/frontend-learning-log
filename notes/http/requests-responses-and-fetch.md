# HTTP 요청·응답과 fetch의 본문 읽기

학습 시작: 2026-10-21 · 마무리 및 기록: 2026-10-22 (Asia/Seoul)
상태: 설명 후 이해와 일부 자기 설명 확인. 마지막 답변의 본문 읽기 시점·변환 결과 표현은 피드백으로 보완했다. 새 조건의 독립 적용과 지연 복습은 미확인.

## 1. HTTP는 무엇을 정하는가

HTTP는 클라이언트와 서버가 요청과 응답을 주고받는 프로토콜이다. 브라우저는 클라이언트의 한 종류이며 모바일 앱이나 다른 서버도 클라이언트가 될 수 있다. 메서드로 요청 의도를 표현하고, 주소로 대상을 지정하며, 응답 상태 코드로 처리 결과를 전달한다. 헤더는 메시지를 해석할 정보를 제공하고 본문에는 실제 데이터를 담는다.

상품 조회를 예로 들면 클라이언트가 “42번 상품 정보를 주세요”라는 요청을 보내고 서버가 상태 코드와 상품 정보를 응답한다. HTTP는 JSON만을 전달하는 규칙이 아니다. HTML, 이미지, 텍스트 등도 전달할 수 있다.

## 2. 요청의 구조와 메서드

일부 헤더를 생략한 HTTP/1.1 요청 예시:

```http
POST /products HTTP/1.1
Host: example.com
Content-Type: application/json
Accept: application/json

{"name":"키보드","price":50000}
```

POST는 처리 의도, /products는 요청 대상이다. Content-Type은 보내는 본문의 형식이 JSON임을 알린다. Accept는 응답으로 받을 수 있는 형식을 알린다. 요청의 Content-Type은 요청 본문, 응답의 Content-Type은 응답 본문을 설명한다. 헤더 아래 빈 줄 뒤의 JSON 텍스트가 본문이다. 헤더에 JSON이라고 적는 것만으로 JavaScript 객체가 문자열로 변환되거나 필수 필드가 검증되는 것은 아니다.

| 메서드 | 의미 | 예시 |
| --- | --- | --- |
| GET | 조회 | 상품 정보 조회 |
| POST | 데이터를 보내 대상이 처리하도록 요청 | 상품 생성, 작업 실행 |
| PUT | 전달한 표현으로 대상 생성·교체 | 전체 정보 교체 |
| PATCH | 부분 변경 적용 | 가격 변경 |
| DELETE | 대상 삭제 | 상품 삭제 |

메서드가 서버 기능을 자동으로 구현하지 않는다. 서버가 해당 주소와 메서드에 맞는 처리를 구현해야 한다. POST를 생성 전용이라고 일반화하지 않는다. 위 표는 설명한 기본 의미이며 멱등성·안전성이나 재시도 설계는 아직 학습하지 않았다.

## 3. 상태 코드와 본문은 역할이 다르다

```http
HTTP/1.1 200 OK
Content-Type: application/json

{"id":42,"name":"키보드","price":50000}
```

200은 요청 처리 결과를 나타낸다. 상품 이름과 가격은 본문에 있다. Content-Type이 JSON이라고 해도 price 필드의 존재나 숫자 타입은 보장하지 않는다. 200 응답이 {"id":42}뿐이라면 price를 사용하는 화면에 필요한 데이터가 부족하다. HTTP 성공, JSON 문법의 유효성, 애플리케이션에서 요구하는 데이터의 유효성은 각각 확인해야 한다.

| 범위 | 의미 |
| --- | --- |
| 1xx | 처리 중 안내 |
| 2xx | 성공 |
| 3xx | 리다이렉션·캐시 사용 등 추가 처리 |
| 4xx | 요청 내용·인증·권한 등의 이유로 처리 불가 |
| 5xx | 서버 측 문제 |

| 코드 | 의미와 사례 |
| --- | --- |
| 200 OK | 요청 성공 |
| 201 Created | 새 자원 생성 성공 |
| 202 Accepted | 접수했으나 처리 완료를 뜻하지 않음 |
| 204 No Content | 성공, 응답 본문 없음 |
| 301 / 302 | 영구적 / 일시적 주소 이동 안내 |
| 304 Not Modified | 조건부 요청에서 기존 캐시 사용 가능 |
| 400 Bad Request | 잘못된 요청 형식 등 |
| 401 Unauthorized | 유효한 인증 필요, 토큰 누락·만료 등 |
| 403 Forbidden | 접근 거부, 권한 부족 등 |
| 404 Not Found | 요청한 자원을 찾지 못함 |
| 409 Conflict | 현재 자원 상태와 충돌 |
| 429 Too Many Requests | 과도한 요청 |
| 500 Internal Server Error | 예상치 못한 서버 내부 오류 |
| 502 Bad Gateway | 중간 서버가 뒤쪽 서버에서 잘못된 응답을 받음 |
| 503 Service Unavailable | 과부하·점검 등으로 현재 서비스 제공 불가 |
| 504 Gateway Timeout | 중간 서버가 뒤쪽 서버 응답을 제때 받지 못함 |

예시의 상태 코드 선택은 실제 API 명세도 확인해야 한다. 위 코드를 설명했지만 학습자가 모두 독립적으로 적용했다고 평가하지 않는다.

404와 500 모두 HTTP 응답이다. 연결 실패 등으로 HTTP 응답을 받지 못한 상황과 구별한다. 중간 서버가 응답할 수도 있으므로 상태 코드를 받았다는 사실이 최종 애플리케이션 서버에 도달했음을 보장하지 않는다.

## 4. fetch로 요청 보내기

```js
const response = await fetch("/products", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    Accept: "application/json",
  },
  body: JSON.stringify({ name: "키보드", price: 50000 }),
});
```

JSON.stringify는 JavaScript 객체를 JSON 문자열로 만든다. 이 예제는 해당 경로를 처리하는 서버가 있는 브라우저 환경을 가정한다. response는 상품 자체가 아니라 응답을 다루는 Response 객체다. response.status로 상태 코드를 확인하며 response.ok는 상태 코드가 200~299이면 true다.

## 5. 왜 await가 두 번 필요한가

```js
const response = await fetch("/products/42");
const product = await response.json();
```

첫 줄은 GET 요청의 응답 상태와 헤더를 사용할 수 있게 될 때까지 기다린다. 본문 전체는 아직 전송 중일 수 있다. 두 번째 줄은 본문을 끝까지 읽고 JSON으로 해석한 JavaScript 값을 기다린다. 상품 예시에서는 객체지만 JSON의 결과가 언제나 객체인 것은 아니다.

```text
상태 코드·헤더 도착 → fetch의 Promise 이행 가능
본문 일부 도착     → {"name":"키보드",
나머지 본문 도착   → "price":50000}
본문 읽기·해석 완료 → json()의 Promise 이행
```

본문은 순서대로 데이터가 들어오는 스트림이다. json()이 Promise를 반환하는 이유를 이해할 때 핵심은 비동기 본문 읽기다. JSON 해석을 별도 스레드에서 처리한다는 뜻은 아니다. 개념상 다음 두 단계로 풀어 볼 수 있다.

```js
const text = await response.text(); // 본문 읽기 → 문자열
const product = JSON.parse(text);  // JSON 문자열 → JavaScript 값
```

이미 문자열이 있다면 JSON.parse는 동기적으로 해석한다. JSON은 텍스트 형식이며, 변환 결과를 “JavaScript 평문”이라고 부르기보다 “JavaScript 값, 이 예시에서는 객체”라고 표현한다.

```js
const text = '{"price":50000}';
const product = JSON.parse(text);
console.log(product.price); // 예상: 50000
JSON.stringify(product);   // 예상 반환값: '{"price":50000}'
```

await만 사용할 수 있는 것은 아니다. json()이 반환한 Promise에 then을 연결해도 된다.

```js
response.json().then((product) => {
  console.log(product.price);
});
```

## 6. HTTP 오류와 Promise 거부는 같은 조건이 아니다

```js
try {
  const response = await fetch("/products/42");
  console.log("성공!");
} catch (error) {
  console.log("실패!");
}
```

500 응답 자체는 fetch의 Promise를 reject시키지 않는다. 위 코드는 500 응답을 받아도 “성공!”을 출력할 수 있다. 네트워크 오류 등으로 요청에 실패하면 reject될 수 있다. HTTP 실패를 오류 흐름으로 보내려면 response.ok 등을 확인하고 직접 throw해야 한다.

```js
async function loadProduct() {
  try {
    const response = await fetch("/products/42");
    if (!response.ok) {
      throw new Error(`HTTP 오류: ${response.status}`);
    }
    const product = await response.json();
    if (
      product === null ||
      typeof product !== "object" ||
      typeof product.price !== "number"
    ) {
      throw new Error("상품 가격 데이터가 올바르지 않음");
    }
    console.log("상품 가격:", product.price);
  } catch (error) {
    console.error("상품을 표시하지 못했어요:", error);
  }
}
```

실행 흐름: 요청 → HTTP 상태 확인 → 본문 읽기·JSON 해석 → 가격 데이터 검사 → 표시. 네트워크 오류, 직접 던진 HTTP 오류, 본문 읽기·해석 오류, 데이터 검사 오류 모두 이 catch로 올 수 있다. catch에 들어왔다는 이유만으로 연결 실패라고 판단하지 않는다.

예제의 한계: 성공 시 JSON 상품을 반환하는 API를 가정한다. 가격 필드의 숫자 여부만 검사하며 모든 상품 데이터나 가격 범위를 검증하는 코드는 아니다. 오류 응답 본문을 읽어 사용자별 메시지로 처리하는 구현도 생략했다. JSON 문법에 맞지 않는 {"price": } 같은 본문은 200이어도 json()에서 실패한다. 본문 없는 204를 무조건 json()으로 읽는 것도 JSON 해석 실패를 일으킬 수 있다.

## 7. 응답 본문은 보통 한 번 소비한다

```js
const response = await fetch("/products/42");
const text = await response.text();
const product = await response.json(); // 예상: 이미 본문을 소비해 실패
```

text()가 스트림을 읽고 나면 json()이 같은 본문을 다시 읽을 수 없다. 문자열과 객체 둘 다 필요하면 읽은 문자열을 재사용한다.

```js
const text = await response.text();
const product = JSON.parse(text);
```

이 내용은 보충 설명 후 이해 응답을 받은 범위다. 독립 적용은 확인하지 않았다.

## 8. 실제 답변과 교정 기록

아래는 학습자의 실제 발언을 기술적 의미에 맞게 요약한 것이며 원문 인용이 아니다.

- 최초: 404는 통신했으나 상품을 찾지 못한 것, 500은 통신 실패라고 설명했다. 500도 HTTP 응답이라는 설명 후 응답은 받았지만 작업은 성공하지 않았다고 정리했다.
- 200이어도 데이터가 올바르게 와야 하며, 가격을 표시하려면 price가 필요하다고 답했다.
- HTTP가 브라우저와 서버의 통신 프로토콜인지 질문했다. 클라이언트 범위를 앱·다른 서버로 넓혀 설명했다.
- 상태 코드 목록 이상의 깊이를 요청하여 요청 구조·메서드·fetch·JSON·오류 처리를 연결해 설명했다. 이후 학습에서도 짧은 확인만으로 종료를 서두르지 않고 원리·문법·실제 사용과 한계를 연결한다.
- json()도 비동기인 이유를 질문하며 JSON을 평문으로 바꾸는 것인지 물었다. 본문 읽기와 JSON 해석을 구분하고 JSON 텍스트 → JavaScript 값의 방향을 설명했다.
- 2026-10-22 최종 답변: 첫 줄은 42번 상품의 GET API 호출을 기다리고, 두 번째 줄은 응답 데이터를 JSON에서 JavaScript 평문으로 바꾸는 작업을 기다리며, 500에서 자동으로 오류가 던져지지 않는다고 설명했다.
- 최종 피드백: 첫 줄의 상태·헤더 시점, 두 번째 줄의 본문 전체 읽기, JavaScript 객체라는 표현을 보완했다. 학습자는 이후 이해했다고 응답했다. 표현만으로 변환 개념 전체를 오해했다고 단정하지 않는다.

자료 없이 정리하도록 안내했지만 실제 참고 여부는 미확인이다. 500과 fetch의 관계는 자기 설명으로 확인했다. 본문 읽기 시점과 결과 표현은 피드백 도움을 받은 이해로 남긴다. 같은 학습을 다음 날 이어 마쳤으며 정식 지연 복습으로 집계하지 않는다. 추가 평가 문제 없이 마무리했다.

## 9. 검증 범위와 다음 확인

이 문서의 예제는 대화에서 설명한 예시이며 실제 실행 검증하지 않았다. 출력·오류는 예상 동작으로 표시했다. 상태 코드별 독립 적용, 오류 분기 구현, 스트림 소비의 독립 적용, 장기 기억 유지는 미확인이다. 복습에서는 HTTP 응답 실패와 네트워크 실패, 두 await의 역할을 먼저 확인한다. 첫 복습 예정일은 마무리 다음 날인 2026-10-23이다.

다음 새 학습은 미학습 영역을 고르게 다루기 위해 TypeScript 타입 추론·유니온·좁히기를 선정한다. 타입과 런타임 데이터 검증의 차이도 오늘의 데이터 확인과 연결할 수 있다.

## 출처

- [MDN: HTTP 개요](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Overview)
- [MDN: HTTP 메서드](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Methods)
- [MDN: HTTP 상태 코드](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status)
- [MDN: Fetch 사용법](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch)
- [MDN: Response.json()](https://developer.mozilla.org/en-US/docs/Web/API/Response/json)
