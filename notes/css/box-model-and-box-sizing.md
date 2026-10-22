# CSS 박스 모델과 box-sizing

학습·기록일: 2026-10-22 (Asia/Seoul)
상태: 당일 학습 마무리. 두 box-sizing의 width 해석을 자기 설명했고 margin 추가 예측을 맞혔다. 마지막 계산 일반화는 피드백으로 보완했으며 보완 후 자기 설명과 지연 기억은 미확인이다.

## 1. 박스의 영역과 width의 기준

요소의 박스는 안쪽부터 content(내용), padding(내용과 테두리 사이 여백), border(테두리), margin(테두리 바깥 여백)으로 구분한다. padding이 테두리 안쪽에 있다는 사실과 CSS width에 포함되는지는 서로 다른 문제다. box-sizing은 지정한 width가 어느 박스의 너비인지를 정한다.

```css
.box {
  box-sizing: content-box;
  width: 200px;
  padding: 20px;
  border: 5px solid black;
}
```

padding에 값 하나를 쓰면 위·오른쪽·아래·왼쪽에 모두 적용된다. border 선언은 사방에 두께 5px의 검은 실선 테두리를 지정한다. 가로 너비에는 왼쪽과 오른쪽만 계산한다.

content-box에서는 내용 영역이 200px이다. 왼쪽 테두리 5 + 왼쪽 padding 20 + 내용 200 + 오른쪽 padding 20 + 오른쪽 테두리 5 = 250px이 테두리까지의 예상 너비다. padding과 border는 지정한 내용 너비에 추가된다.

## 2. border-box는 포함 범위를 바꾼다

같은 코드에서 box-sizing만 border-box로 바꾸면 width 200px은 테두리까지 포함한 너비다. padding이나 border를 없애는 것이 아니라 그 공간을 200px 안에서 사용한다. 따라서 내용 영역은 200 − 20 − 20 − 5 − 5 = 150px이다.

| 설정 | 내용 영역 | 테두리까지 너비 |
| --- | ---: | ---: |
| content-box | 200px | 250px |
| border-box | 150px | 200px |

box-sizing의 초기값은 content-box다. 프로젝트에서 공통 CSS로 border-box를 적용할 수 있으므로 초기값과 프로젝트의 실제 적용값은 구분한다. 보충 설명: 브라우저 기본 스타일이 일부 폼 요소 등에 border-box를 적용하기도 하므로 모든 요소의 실제 값이 언제나 content-box라는 뜻은 아니다.

지정한 전체 크기를 유지하면서 여백과 테두리를 넣고 싶을 때 border-box가 유용하다. 사용 기준은 에이전트가 설명했고 최종 답변에서 학습자가 별도로 자기 설명한 것은 아니다.

## 3. 부모 너비와 100% 예제

```html
<div class="parent">
  <div class="child">내용</div>
</div>
```

```css
.parent { width: 300px; }
.child {
  width: 100%;
  padding: 20px;
  border: 5px solid black;
  box-sizing: content-box;
}
```

다른 크기 제약이 없는 일반적인 블록 배치를 가정한다. 이 예제에서 자식의 100%는 부모 내용 영역의 300px을 기준으로 한다. content-box에서는 내용만 300px이므로 좌우 padding 40px과 border 10px을 더하면 테두리까지 350px이다. 부모 내용 영역보다 커진다.

border-box로 변경하면 테두리까지 300px이고 내용은 250px이다. 이 계산은 에이전트의 설명이며 학습자의 독립 예측으로 집계하지 않는다.

## 4. margin과 계산 표현

margin은 두 box-sizing 모두 width에 포함되지 않는다. 위 자식이 border-box이고 margin: 10px을 추가하면 지정한 테두리까지의 너비 300px에 좌우 margin 20px을 더한 산술 합은 320px이다. CSS width 자체가 320px로 바뀌는 것은 아니다.

학습자는 margin이 바깥에 추가되어 320px이라고 예측했다. 에이전트는 이를 테두리까지 너비와 margin을 더한 값으로 명확히 했다.

가로 계산은 항상 필요한 영역의 왼쪽 값과 오른쪽 값을 각각 더한다. 두 배 계산은 양쪽이 같을 때의 줄임 표현이다. 예를 들어 padding-left: 10px, padding-right: 30px이면 합은 40px이다. 세로 크기를 계산할 때는 위·아래를 구분한다.

- content-box: 테두리까지 너비 = width + 좌우 padding + 좌우 border.
- border-box: 테두리까지 너비 = width. 내용 너비를 구하려면 좌우 padding과 border를 뺀다.
- 두 방식 모두: 위 예제의 margin 포함 산술 합 = 테두리까지 너비 + 좌우 margin.

보충 설명과 예제 한계: 여기서는 양수 고정 여백과 충분한 width를 사용했다. border-box라도 padding과 border의 합보다 작게 지정하면 내용 영역은 음수가 되지 않으므로 테두리까지의 실제 너비가 지정값을 넘을 수 있다. 일반 블록의 과잉 제약 시 사용 margin 조정, auto·음수 margin, 세로 margin 겹침, flex/grid 크기 조정은 이번 학습에서 다루거나 평가하지 않았다. 320px은 이번 질문의 지정값 합이며 모든 배치에서 실제 사용 margin이나 스크롤 너비를 측정한 결과가 아니다.

## 5. 실제 답변과 교정

아래는 원문 인용이 아닌 기술적 의미의 요약이다.

1. 최초에는 content-box에서 padding과 border가 width 안에 포함되므로 전체 200px이라고 답했다. 설명 후 border-box와 혼동했다고 스스로 밝혔다.
2. content-box는 내용만 width이고 padding과 border가 추가된다고 정리한 뒤 기본값을 질문했다.
3. border-box에서 영역의 너비를 200px이라고 답했다. 전체와 내용 영역을 명확히 나눈 설명 후 내부가 150px이라고 확인했다. 이 과정은 피드백 후 이해로 기록한다.
4. 부모 300px, 자식 border-box의 width 100%에 좌우 margin 10px을 더한 변형에서 320px이라고 예측했다. 추가 힌트 없이 맞힌 제한된 적용 근거다. 실제 자료 참고 여부는 미확인이다.
5. 최종 정리에서 content-box는 지정 크기에 padding·border가 추가되고 border-box는 지정 크기 안에 포함된다고 정확히 설명했다. padding·border·margin을 모두 두 배로 계산해 최종 width를 구한다는 표현은 보완했다. 가로는 좌우를 더하며 동일할 때만 두 배이고, border-box의 padding·border는 width에 다시 더하지 않는다. margin 포함 크기와 CSS width도 구별한다.

마지막 정리는 앞선 설명 없이 시도하도록 안내했으나 실제 참고 여부는 미확인이다. 최종 보완 후 자기 설명은 아직 받지 않았다. 같은 날 추가 시험을 붙이지 않고 학습을 마친다. 정식 복습 점수나 반복 오류 횟수를 추가하지 않는다.

## 6. 검증 범위와 다음 질문

계산은 설명을 위한 예상 결과다. 브라우저 렌더링이나 개발자 도구를 통한 실행 검증은 하지 않았다. 서로 다른 좌우 값에서의 독립 계산, 실제 레이아웃 수정, 지연 기억은 미확인이다. 첫 복습 후보일은 2026-10-23이며 두 box-sizing 및 margin의 계산 기준을 다시 확인한다. 다음 새 주제는 실무 기본기를 위한 재현과 브라우저 개발자 도구 디버깅이다.

## 출처

- [W3C CSS Box Sizing Module Level 3 — box-sizing](https://www.w3.org/TR/css-sizing-3/#box-sizing): 크기 기준과 내용 영역의 하한.
- [MDN — box-sizing](https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Properties/box-sizing): 기본값, 문법과 두 설정의 비교.
- [MDN — The box model](https://developer.mozilla.org/en-US/docs/Learn_web_development/Core/Styling_basics/Box_model): content·padding·border·margin 구분.
