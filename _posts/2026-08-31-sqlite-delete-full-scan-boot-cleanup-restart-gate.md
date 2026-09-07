---
title: "삭제 0건인데 6.4초 — SQLite <> 조건의 풀스캔과 재시작마다 반복되는 정리 배치"
excerpt: "에이전트 시작 시 이전 부팅 데이터 정리 배치 다섯 쿼리가 모두 삭제 0건인데 6.4초를 썼다. WHERE BootUID <> ? 가 SQLite에서 인덱스를 타지 않아 44,210행 테이블을 콜드 캐시에서 전량 스캔한 것이 원인이었다. <> 를 < OR > 로 재작성해 커버링 인덱스로 바꾸고, 같은 부팅의 재시작에서는 정리를 건너뛰는 게이트를 신설한 과정."
category: tech
date: 2026-08-31
author: kim-tigerj
tags: [SQLite, 쿼리최적화, 풀스캔, 인덱스, Windows에이전트, 트러블슈팅, Orange Platform]
---

## 현상

에이전트(orange.exe) 시작 시 이전 부팅 데이터 정리 배치가 오래 걸린다. 개발 PC 실측(2026-08-31, summary.cdb 46.7MB, 5400rpm HDD):

```
   2.2 ms  SPROCESS_DELETE_OLD (0)
 771.1 ms  SDATA_DELETE_OLD (0)
   0.0 ms  DETECT_DELETE_OLD (0)
   0.8 ms  REPORT_DELETE_OLD (0)
5596.4 ms  TIMELINE_DELETE_OLD (0)
```

다섯 쿼리 모두 삭제 0건인데 6.4초를 썼다. 재부팅 없이 에이전트만 재시작한 경우라 지울 행이 없었다.

## 원인

두 가지가 겹쳤다.

- `DELETE from timeline WHERE BootUID <> ?`는 SQLite가 `<>` 조건에 인덱스를 쓰지 않아 전체 테이블 스캔이 된다. 실제 DB에서 `EXPLAIN QUERY PLAN`으로 `SCAN TABLE timeline` 확인. `timeline.idx.BootUID` 인덱스가 있어도 타지 않는다.
- timeline은 부팅 세션 안에서 계속 쌓인다. 이 PC는 8/28 부팅 후 3.5일 동안 44,210행(평균 텍스트 402바이트)이 쌓였고, 시작 직후라 OS 캐시가 비어 있어 HDD에서 테이블 전체를 읽었다.

소요 시간은 테이블 크기에 비례한다. sprocess(1,585행) 2.2ms, sdata(10,600행) 771ms, timeline(44,210행) 5,596ms.

## 조치

수정 파일: `db/summary.json`, `yagent21/CAgentHelper3.cpp`

**TIMELINE_DELETE_OLD 쿼리 재작성**: `WHERE BootUID <> ?` → `WHERE BootUID < ? OR BootUID > ?`. 결과는 같고(동일 값 두 번 바인딩), `EXPLAIN QUERY PLAN`이 `MULTI-INDEX OR` + `COVERING INDEX timeline.idx.BootUID`로 바뀐 것을 실제 DB에서 확인했다. 지울 행이 없으면 인덱스 탐색만으로 끝난다.

**재시작 게이트 신설**: 정리 배치 전에 `AGENT_COUNT_THISBOOT`(신규 쿼리)로 이번 부팅의 agent 실행 기록 수를 센다. 기록이 있으면 같은 부팅의 이전 실행이 이미 정리를 마친 것이므로 부팅 단위 정리 4종(SPROCESS·SDATA·REPORT·TIMELINE)을 건너뛴다. DETECT_DELETE_OLD는 30일 보존 조건이 있어 항상 실행한다. agent 실행 기록은 정리 배치보다 뒤에 기록되므로, 이전 실행이 정리 도중 죽었으면 기록이 없어 다시 정리한다.

기대 효과: 재시작(이번 사례) 시 6.4초 → 수 ms. 재부팅 시에는 실제 삭제가 일어나며 timeline 스캔이 인덱스로 바뀐다.

## 검토 후 제외한 대안

- **재부팅 시 `DELETE FROM timeline`(WHERE 없는 truncate 최적화)**: `CProcessCallback2` 등 다른 컴포넌트가 이 배치보다 먼저 현재 부팅 행을 넣는 경로가 있어, 새 행까지 지울 위험이 있다.
- **sdata에 BootUID 인덱스 추가**: 게이트로 재시작 스캔이 사라지면 남는 비용은 재부팅 1회의 실제 삭제뿐이다. 인덱스는 sdata를 쓸 때마다 비용을 얹는다.

## 남은 결정

교차검증에서 나온 지적: 게이트는 "이전 실행이 정리 코드를 통과했다"는 증거일 뿐 "삭제가 커밋됐다"는 증거는 아니다. 삭제나 커밋이 실패해도 agent 기록은 남아 그 부팅 동안 계속 건너뛴다. 다만 실패해도 조회는 BootUID 필터라 기능 영향이 없고, 다음 재부팅에서 다시 정리된다. 성공 마커를 따로 기록하는 보강은 구조 변경이 커서 반영하지 않았다.

timeline이 부팅 세션 안에서 상한 없이 자라는 문제는 별도 결정이 필요하다. 재부팅을 오래 안 하는 PC에서는 이번처럼 수만 행이 된다.

*Orange Platform 엔드포인트 에이전트의 시작 정리 배치 분석 리포트입니다.*
