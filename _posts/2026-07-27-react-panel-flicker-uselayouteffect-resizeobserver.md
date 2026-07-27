---
title: "React 위젯 첫 진입 시 패널이 깜빡이는 이유 — ResizeObserver는 paint 이후에 실행된다"
excerpt: "설정 위젯 최초 진입 시 좌측 탭 네비가 한 번 깜빡이는 문제를 프레임 단위로 추적한 기록. 패널 폭을 ResizeObserver로 실측했는데 그 콜백이 paint 이후에 호출돼, 잘못된 폭으로 그려진 프레임 + key 변경 강제 리마운트가 겹쳐 깜빡임이 발생하던 원인을 규명하고 useLayoutEffect 동기 실측으로 해결했다."
category: tech
date: 2026-07-27
author: hja
tags: [React, useLayoutEffect, ResizeObserver, 렌더링 타이밍, react-resizable-panels, 프론트엔드 성능, Playwright, Orange Platform]
---

## 증상

설정 위젯에 처음 진입할 때 좌측 탭(카테고리 네비) 영역이 한 번 깜빡인다. 패널이 넓게 그려졌다가 순간적으로 좁아지고, 그 뒤에 탭 항목들이 페이드인된다.

## 원인

`src/components/widgets/setting/index.tsx`의 좌측 네비 패널 폭 초기화 순서 문제다.

`react-resizable-panels`는 % 단위만 받으므로 기준 px(기본 300px, 저해상도 220px)을 컨테이너 실측 폭으로 환산해야 한다. 그런데 실측을 **ResizeObserver에서 수행했고, ResizeObserver 콜백은 paint 이후에 호출된다.** 결과적으로 아래 순서가 된다.

1. **첫 페인트** — `panelSizes`가 실측 전 하드코딩 초기값 `{ default: 25, min: 15, max: 50 }`(%) → 네비가 잘못된 폭으로 그려짐
2. **paint 후** ResizeObserver가 실측 → `setPanelSizes()` + `setPanelGroupKey(k + 1)`
3. key 변경으로 `PanelGroup` 전체가 언마운트/리마운트 → 폭이 튀고, 선택된 탭 항목의 진입 애니메이션(`opacity 0→1` + `translateX 10px`)이 그 시점에 재생

즉, 잘못된 폭으로 그려진 프레임 + 강제 리마운트가 겹쳐 깜빡임으로 보인다.

## 재현 계측 (Playwright headless, requestAnimationFrame 프레임 단위 폭 샘플링)

| 조건 | 첫 페인트 폭 | 폭 변화 |
|------|-------------|---------|
| 수정 전 (1920×1080) | 414px → 약 100ms(6프레임) 뒤 299px | 2단계 점프 |
| 수정 후 (1920×1080) | 298px | 없음 |
| 수정 후 (1440×900, 저해상도) | 218px | 없음 |

계측 방법: 홈 → 설정 전환 직전에 rAF 루프로 `[data-panel].bg-gray-100.overflow-hidden`의 `getBoundingClientRect().width`를 매 프레임 기록하고, `animationstart` 이벤트로 진입 애니메이션 재생 횟수를 카운트했다.

## 수정 내용

- **`useLayoutEffect`로 paint 전에 동기 실측** — `getBoundingClientRect()`로 컨테이너 폭을 재서 `panelSizes`를 확정한다. ResizeObserver와 달리 브라우저가 그리기 전에 실행되므로 잘못된 폭의 프레임이 생기지 않는다.
- **`panelSizes` 초기값을 `null`로 변경** — 폭이 확정되기 전에는 `PanelGroup`을 아예 렌더하지 않는다. 임의 기본값(25%)으로 그려지는 프레임 자체를 제거.
- **ResizeObserver의 최초 확정 분기에서 `setPanelGroupKey` 제거** — `PanelGroup`이 폭 확정 후에만 마운트되므로 리마운트가 불필요하다. 이 분기는 위젯이 폭 0인 숨겨진 상태로 마운트되는 경우의 폴백 경로로만 남긴다.
- **저해상도 전환 effect에 `prevLowResRef` 가드 추가** — 위 변경으로 초기화 플래그가 마운트 시점에 이미 true가 되므로, 기존 가드만으로는 마운트에서도 이 effect가 통과해 `panelGroupKey`를 올려 깜빡임이 되살아난다. 실제 1600px 경계를 넘는 전환일 때만 동작하도록 이전 값과 비교한다.

## 회귀 확인

- 드래그 리사이즈 정상 (min 160px / max 560px)
- 창 크기 변경 시 min/max 재환산 유지
- 1600px 경계 전환 시 기본 폭 300 ↔ 220 전환 정상 (1920→1440 리사이즈 후 실측 218px)
- 저해상도(1440×900) 첫 진입 폭 218px, 폭 변화 0회
- 콘솔 에러 0건, `tsc --noEmit` 신규 에러 없음, eslint 통과

## 변경 파일

- `src/components/widgets/setting/index.tsx`

---

*이 글은 Orange Platform 매니저 웹의 렌더링 타이밍 분석 및 트러블슈팅 리포트입니다.*
