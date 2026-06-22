# 엔진 설계 배경 & 브라우저 호환성

`assets/template.html` 엔진이 왜 이렇게 짜였는지에 대한 근거. 엔진을 수정할 때 참고한다.

## 핵심 설계 철학

> **고정 기준 해상도(1280×720)로 디자인 → `transform: scale()`로 뷰포트에 맞춰 균일 스케일링**
> **+ 단일 상태(`current`, `stepState[]`, `isAnimating`)를 `goTo()`/`applyStep()` 한 곳에서만 변경**

모든 입력(방향키 / 스와이프 / hash)은 결국 `goTo(index)` 하나로 수렴한다. 상태 변경 지점을
한 곳으로 모으면 진행바·번호·hash 동기화가 어긋날 일이 없다.

## 왜 이렇게 했나 (주요 결정)

| 결정 | 이유 |
|---|---|
| 슬라이드를 `position:absolute`로 겹치고 `.active`만 노출 | `display:none`은 CSS 트랜지션이 안 걸린다. opacity/visibility 토글로 부드러운 전환. |
| `transform`+`opacity`만 애니메이트 | GPU 합성 레이어에서 처리되어 layout/paint를 건너뛴다(부드러움). |
| `transform: scale()`로 16:9 고정 | `aspect-ratio`보다 호환성이 넓고(전 브라우저), 픽셀 고정 레이아웃이라 깨짐이 없다. |
| `Math.min(가로비, 세로비)` | 잘림 없는 letterbox. 남는 여백은 컨테이너 배경색이 채운다. |
| `isAnimating` 플래그 + `transitionend` + setTimeout 폴백 | 전환 도중 입력이 겹쳐 슬라이드가 꼬이는 것을 막는다. `transitionend`가 누락돼도 폴백으로 잠금 해제. |
| `height:100vh` 다음 줄에 `100dvh` | 모바일 주소창 대응(dvh). 미지원 구형은 앞줄 vh로 폴백. |
| 키보드에서 `event.key` 사용 + `isComposing` 가드 | Space의 key는 `' '`. 한글 IME 입력 중복 입력 방지. |
| hash는 `replaceState` + `#slide-N` | 새로고침/딥링크 진입 지원. 숫자만 쓰면 일부 브라우저가 앵커 점프로 처리하므로 접두사 사용. |

## 브라우저 호환성 메모

- `transform:scale` — 전 브라우저 완전 지원(가장 안전).
- `dvh/svh/lvh` — Chrome/Edge 108+, Firefox 101+, Safari 15.4+ (현재 Baseline). `vh` 폴백 필수.
- `aspect-ratio`(엔진은 미사용) — Safari는 15+부터. 그래서 엔진은 scale 방식을 택했다.
- Fullscreen API — `requestFullscreen()`는 **사용자 제스처(키 입력) 핸들러 안**에서 호출해야 하며,
  일부 브라우저는 프리픽스/거부가 있어 `?.()` + `.catch()`로 방어한다. `file://`에서도 동작한다.
- highlight.js / line-numbers 플러그인 — CDN 로드. 오프라인이면 코드 색상이 빠지지만 슬라이드는 정상 동작.

## 흔한 함정

- `100vw`는 세로 스크롤바 폭을 포함 → 가로 스크롤 유발. 컨테이너는 `width:100%`.
- `will-change`를 모든 요소에 영구 선언하면 레이어 폭증/메모리 과다. 전환 대상에만 제한.
- 코드를 `innerHTML`로 토큰화하면 `<`,`&` 깨짐 + XSS. highlight.js에 맡긴다.
- `transitionend`는 속성마다 여러 번 발생 → `e.propertyName` 필터 필요.

더 깊은 리서치(전환 애니메이션 비교, 화자 노트/Presenter View, 터치 상세 등)는 프로젝트 루트의
`RESEARCH.md`에 정리되어 있다.
