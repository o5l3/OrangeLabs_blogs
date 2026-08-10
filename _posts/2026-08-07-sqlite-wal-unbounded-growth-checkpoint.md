---
title: "SQLite WAL 무한 성장 잡기 — 체크포인트 비활성 결함과 zipvfs 제약 하의 크기 감시 설계"
excerpt: "자동 체크포인트가 꺼진 WAL 모드 SQLite가 디스크를 고갈시킨 결함을, 실측·zipvfs 제약·주기 체크포인트·크기 감시 설계로 해결한 트러블슈팅 기록."
category: tech
date: 2026-08-07
author: kim-tigerj
tags: [SQLite, WAL, zipvfs, 체크포인트, Windows에이전트, Orange Platform]
---

## 1. 개요

앞선 에이전트 집단 종료 사건 조사 중, 디스크 여유가 0바이트인 테스트 노드에서 `summary.cdb`(본체 21MB)의 **WAL 파일이 295MB까지 부푼 것**을 발견했다. 조사 결과 에이전트 로컬 DB 4개(summary/current/config/log) 전부에서 WAL이 상한 없이 단조 증가하는 구조적 결함을 확정했다. 이 글은 원인, 실측, 확정 설계, 코드 개선안을 정리한다.

## 2. 원인 (확정)

- 4개 DB 모두 WAL 모드인데 **자동 체크포인트가 꺼져 있다**: `db/summary.json`·`db/db.json`에 `"wal_autocheckpoint": 0` 명시, current/config/log는 필드가 없어 코드 기본값 0 적용(`CDbClass2.cpp:2301`). 0으로 둔 것은 수집(쓰기) 경로의 체크포인트 순간 지연을 피하려는 의도였다.
- **수동 체크포인트 호출도 전무**(`PRAGMA wal_checkpoint` 실행 0건). 따라서 WAL이 정리되는 유일한 시점은 에이전트 정상 종료뿐이다.
- 결과: 장기 실행 노드에서 WAL 무한 성장, 비정상 종료 시 거대 WAL 잔존. 디스크 고갈에 가담(사고 노드에서 실측).
- **zipvfs 제약**: C API `sqlite3_wal_checkpoint()`는 무효, `PRAGMA wal_checkpoint`/`wal_autocheckpoint` 문장만 하위 페이저로 라우팅된다(zipvfs readme).

## 3. 실측

| 항목 | 값 |
|------|-----|
| 하위 페이저 페이지 크기 | 4,096B (WAL 헤더 실측), 프레임 = 4,120B |
| WAL 성장률 — summary | 1.4~3.6MB/h (개발 PC~테스트 VM) |
| WAL 성장률 — current | 9.4MB/h (최대 기록원) |
| 사고 노드 | `summary.cdb-wal` 295MB / 약 81시간 |

## 4. 확정 설계

- **평시**: DB별 워커 큐에 5분 주기 `PRAGMA wal_checkpoint(PASSIVE)` (1회 처리량 최대 ~800KB, 밀리초~수십ms). 잡 중복 방지(DB당 pending 1개).
- **크기 감시(에이전트 자신)**: 체크포인트 직후 `-wal` 파일 크기 직접 확인. 16MB 초과 → 한가한 틈에 `TRUNCATE` 시도. 64MB 초과 → 재시도 간격 단축 + 진단 로그.
- **안전망**: `wal_autocheckpoint=2000`(약 8.24MB, current 성장률 기준 도달 ~50분). 주기 잡이 못 돌 때만 발동.
- **시작 시 대청소**: 초기화 직후 짧은 타임아웃(100~500ms)으로 `PRAGMA wal_checkpoint(TRUNCATE)` 1회 — 비정상 종료 잔존 WAL을 파일 크기까지 회수. 실패해도 시작을 막지 않고 워커에서 30초→2분→10분 백오프 재시도.

### 교차검증 반영 (조건부 승인 → 보완)

- **PASSIVE 성공 ≠ 완전 회수**: 장수명 리더가 있으면 부분 복사 후 정상 반환. 사용 중인 zipvfs 3.50.4는 체크포인트 PRAGMA 결과를 단일 값(0=OK, 1=BUSY)으로 축약 — 3값 통계가 오지 않으므로 판정은 **반드시 `-wal` 파일 크기로** 한다.
- **TRUNCATE의 BUSY는 SQL 오류가 아니라 결과 행 1** — 실행 성공만 보면 실패를 놓친다. 결과 행 판독 필수.
- **`journal_size_limit` 제외**: zipvfs 하위 페이저 적용이 소스상 보장되지 않음. 회수는 `TRUNCATE`로 일원화.
- **SQLite 3.50.4 WAL-reset 손상 버그(별도 과제)**: 서로 다른 커넥션의 동시 쓰기/체크포인트에서 드물게 손상 가능, 3.50.7에서 수정. 현 구조(DB당 단일 워커 직렬화)는 발동 조건이 낮으나 zipvfs 3.50.7+ 업그레이드 검토 권고.
- 디스크가 이미 0인 상태의 체크포인트는 복구 수단이 못 됨(`SQLITE_FULL`) — 본 설계는 **예방** 목적. 디스크 여유 임계 미만 시 수집 억제(백프레셔)는 후속.

## 5. 코드 개선안 (변경 범위)

| 파일 | 변경 |
|------|------|
| `db/summary.json`, `db/current.json`, `db/config.json`, `db/log.json` | `"wal_autocheckpoint": 2000` 명시 (db.json은 빌드 시 mergedb가 재생성 → DB JSON 리소스로 내장) |
| `module/CDbClass2.h/.cpp` | `CheckpointWal(bTruncate, busyTimeoutMs)` API 추가 — PRAGMA 실행 + 결과 행(0/1) 판독 + `-wal` 파일 크기 반환, db 로그에 크기·소요·결과 기록 |
| `yagent21/CAgent2.cpp` | 메인루프 5분 자리(`dwCount % 300`)에서 DB별 체크포인트 잡 투입(중복 방지), 16/64MB 단계 대응, 초기화 직후 `TRUNCATE` 1회 + 백오프 재시도 |

> 주의: dev 모드는 기존대로 `wal_autocheckpoint=1` 강제 유지(`CDbClass2.cpp:2308`). 스키마 JSON은 `orange.exe` 리소스로 내장되므로 설정 반영에는 리빌드가 필요하다.

---

*Orange Platform 엔드포인트 에이전트의 로컬 DB 안정화 트러블슈팅 리포트입니다.*
