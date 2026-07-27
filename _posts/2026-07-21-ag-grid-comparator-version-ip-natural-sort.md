---
title: "AG Grid에서 IP·버전이 뒤집혀 정렬되는 이유 — comparator 부재로 인한 사전식 정렬 전역 수정"
excerpt: "그리드의 버전 컬럼이 x.x.x.x 형식임에도 문자열로 정렬된다는 의심에서 출발해, 코드베이스에 AG Grid comparator가 하나도 없어 모든 정렬 컬럼이 기본 사전식 비교에 의존하던 구조적 문제를 밝혀낸 기록. 192.168.1.100이 192.168.1.20보다 앞서는 IP 오정렬과 버전 오정렬을 옥텟·세그먼트 숫자 비교자로 전역 교정했다."
category: tech
date: 2026-07-21
author: smahn9123
tags: [AG Grid, comparator, 자연 정렬, IPv4, 버전 정렬, React, TypeScript, Orange Platform]
---

## 개요

"전체 노드" 위젯의 버전 컬럼이 `x.x.x.x` 형식임에도 단순 문자열로 정렬된다는 의심에서 출발했다. 확인 결과 사실이었고, 원인을 추적하니 개별 컬럼의 실수가 아니라 코드베이스 전역의 구조적 문제였다. **이 저장소에는 AG Grid comparator가 단 하나도 정의돼 있지 않아, 모든 정렬 컬럼이 AG Grid 기본 비교자(문자열 사전식)에 의존하고 있었다.** 그 결과 사전식 순서와 의미적 순서가 다른 데이터(버전·IP)가 전부 오정렬됐다. 총 20개 컬럼 + OS 버전 트리 1곳을 수정했다.

## 기존 vs 변경 흐름 비교

```
[기존]
colDef { sortable: true, valueGetter: () => '192.168.1.100' }
   └─ comparator 없음 → AG Grid 기본 비교자 = 문자열 사전식
        192.168.1.100 < 192.168.1.20 < 192.168.1.3    ← 뒤집힘
        192.168.9.4   > 192.168.88.3                  ← 뒤집힘
        1.6.242.100   < 1.6.242.97                     ← 뒤집힘
        10.0.22631    < 10.0.9600   (OS 버전)          ← 뒤집힘

[변경]
colDef { sortable: true, valueGetter: ..., comparator: compareNodeIp }
   └─ 옥텟/세그먼트를 숫자로 비교
        192.168.1.3 < 192.168.1.20 < 192.168.1.100    ← 정상
        192.168.9.4 < 192.168.88.3                    ← 정상
        1.6.242.97  < 1.6.242.100                      ← 정상
        10.0.9600   < 10.0.22631                       ← 정상
```

## 변경 범위

신규 생성 (1)

| 파일 | 역할 |
|------|------|
| `src/utils/version.ts` | `compareVersion` — 점 구분 버전(에이전트·파일·OS)을 세그먼트별 숫자 비교. AG Grid comparator 및 `Array.sort` 콜백 양쪽으로 사용 가능 |

수정 (16)

| 파일 | 변경 내용 |
|------|-----------|
| `src/utils/nodeIp.ts` | `compareNodeIp` 추가 — IPv4 옥텟 숫자 비교. 기존 "노드 IP 표시 단일 정본" 모듈에 배치 |
| `widgets/nodes/Grid.tsx` | IP, 에이전트 버전 |
| `widgets/nodes/index.tsx` | OS 그룹핑 트리 자식 정렬 (colDef 아님 — 순수 JS `localeCompare`) |
| `widgets/organization/NodeGrid.tsx` | IP, 에이전트 버전 |
| `widgets/unconnectedNodes/Grid.tsx` | IP, 에이전트 버전 |
| `widgets/virtualGroups/Grid.tsx` | IP, 에이전트 버전 |
| `widgets/liveCmdLogs/Grid.tsx` | 에이전트 IP, 에이전트 IP2, 관리자 IP |
| `widgets/nodesLoad/Grid.tsx` | IP |
| `widgets/nodesLoad/AutoUpdateGrid.tsx` | IP |
| `widgets/detectsNodes/Grid.tsx` | IP |
| `widgets/summaryNodes/Grid.tsx` | IP |
| `widgets/nodesByNodeId/Grid.tsx` | IP(내부) — valueGetter 추가 + comparator |
| `widgets/nodeAnalysis/index.tsx` | IP |
| `widgets/nCommandMain/components/ResultGrid.tsx` | IP |
| `widgets/nOrderManagement/components/ResultGrid.tsx` | IP |
| `widgets/causeGroupedDetectType/Grid.tsx` | 파일 버전 |

## 주요 변경 사항

- **IP 정렬 14개 컬럼(12개 파일) 교정 — 버전보다 심각하다.** 버전은 빌드 번호가 세 자리가 돼야 터지는 잠재 버그지만, IP는 평범한 /24 대역에서 지금 당장 깨진다. `192.168.1.100 → 192.168.1.20 → 192.168.1.3` 순으로 뒤집히고, `192.168.9.4`가 `192.168.88.3`보다 아래로 간다.
- `organization/NodeGrid.tsx`는 사용자가 아무것도 누르지 않아도 틀린 순서로 열리고 있었다 — IP 컬럼에 `sort: 'asc'`가 걸려 있어 노드 할당 다이얼로그의 기본 정렬이 오정렬 상태였다. 체감 영향이 가장 큰 지점이다.
- **버전 정렬 6곳 교정** — 에이전트 버전 4곳, 파일 버전 1곳, OS 버전 트리 1곳. OS 버전 트리는 colDef가 아니라 순수 JS `localeCompare` 정렬이라 comparator를 받을 수 없어, 공용 함수를 직접 호출하도록 했다. 이것이 헬퍼를 `src/utils/version.ts`로 승격한 이유다.
- **부수 결함 1건 — 표시/정렬 키 불일치.** `nodesByNodeId`는 셀이 `ip || ip2`를 표시하는데 정렬 키는 `field: 'ip'` 단독이라, `ip2`로 표시되던 행들이 빈 값으로 정렬되고 있었다. comparator만으로는 해결되지 않아 valueGetter를 표시 로직과 일치시키고 cellRenderer가 `params.value`를 쓰도록 했다.
- **안전 확인** — 숫자·날짜·크기 컬럼은 손대지 않았다. 이 코드베이스는 `valueGetter → Number + valueFormatter → 표시` 관용구를 일관되게 지키고 있어 파일 크기·퍼센트·카운트 컬럼은 이미 정상이다. 날짜 컬럼도 ISO / `YYYY-MM-DD HH:mm:ss` 키라 사전식 == 시간순으로 안전하다.

## 주의사항 / 리뷰 포인트

- 빈 값·비정상 값 정책 — 미수집 IP(`''`), 옥텟 부족, 비숫자 세그먼트는 모두 최하위로 정렬된다. 기존 문자열 정렬에서 빈 문자열이 최하위였던 것과 같은 순서라 회귀가 없다.
- `compareNodeIp`는 옥텟 값을 검증하지 않는다(`999.1.1.1`도 통과). 정렬 전용이며 유효성 검사가 목적이 아니다.
- `buildGroupingTree`는 os/device 모드가 공유하는 함수다. 자식 키가 OS 버전일 때만 버전 비교여야 하고 장비 모델명은 사전식이 맞아, `mode === 'os'` 분기를 두었다. 이 분기를 제거하면 장비 모델명 정렬이 깨진다.
- **이 저장소의 첫 comparator 도입이다.** 앞으로 정렬 컬럼을 추가할 때 "valueGetter가 문자열을 반환하는데 의미적 순서가 사전식이 아닌가"를 점검 항목으로 삼는 것이 좋다.

## 검증 결과

| 항목 | 결과 |
|------|------|
| `tsc --noEmit` | 변경 파일 관련 에러 0 |
| eslint | 신규 에러 0 (잔존 2건은 수정 전 원본에서도 재현되는 기존 문제) |
| vite build | 통과 (1m 20s) |
| 단위 단언 | 두 비교자 15개 케이스 전부 통과 (실제 화면 값 + 경계값: 빈칸·옥텟 부족·비숫자·동일 값) |

---

*이 글은 Orange Platform 매니저 웹 그리드 정렬 로직 분석 및 트러블슈팅 리포트입니다.*
