---
title: "부팅으로 가장 바쁜 순간에 얹히던 WAL 체크포인트 1.4초 — 주기 루프 첫 틱을 건너뛰다"
excerpt: "SQLite WAL 체크포인트를 5분 주기로 도는 루프가 카운터 0에서 시작해 첫 틱에도 즉시 발동했다. 시작 폭풍이 WAL에 쓴 분량을 HDD에서 곧바로 회수하느라 부팅 직후에 디스크 I/O 1.4초를 얹던 문제를, 같은 파일의 다른 주기 잡이 이미 쓰던 '실행 직후 최초 시점엔 실행 안 함' 방어를 체크포인트 블록에도 넣어 해결한 과정."
category: tech
date: 2026-09-01
author: kim-tigerj
tags: [SQLite, WAL, 체크포인트, Windows에이전트, 시작지연, 트러블슈팅, Orange Platform]
---

## 현상

에이전트(orange.exe) 시작 수 초 뒤 WAL 체크포인트 작업이 느리게 돈다. 개발 PC 실측(2026-08-31, 5400rpm HDD):

```
22:18:17.765306  SLOW JOB  CheckpointDatabases  1.3678824(s)
```

작업 큐(비동기)라 시작 자체를 막지는 않지만, 부팅·로그온으로 가장 바쁜 순간에 디스크 I/O 1.4초를 얹는다.

## 원인

5분 주기 WAL 체크포인트가 주기 루프의 첫 틱에도 발동한다.

- `RunLoop`의 카운터가 0에서 시작하고, 주기 조건이 `0 == dwCount % 300`이라 루프 첫 틱에 즉시 `CheckpointDatabases`가 큐에 들어간다(`CAgent2.cpp RunLoop`).
- 시작 대청소(비정상 종료 잔재 TRUNCATE 회수)는 몇 초 전 `InitializeDB`에서 이미 돌았다.
- 그 몇 초 사이 시작 폭풍(프로세스 전수 열거 커밋, agent 기록, 정리 배치)이 WAL에 쓴 분량을 첫 틱 체크포인트가 HDD에서 곧바로 다시 회수하느라 1.368초가 걸렸다.

같은 파일의 `CheckUpdate` 주기는 `if (dwCount && ...)`로 "실행 직후 최초 시점엔 검사 안 함"을 명시해 두었는데, 체크포인트 블록에만 이 방어가 없다.

## 조치

`CAgent2.cpp RunLoop`의 체크포인트 등록에 첫 틱 제외 조건을 추가한다. `if (0 != dwCount)`일 때만 `CheckpointDatabases`를 큐에 넣는다. `CheckUpdateFiles`는 기존 동작을 유지한다.

첫 실행을 5분 뒤로 미뤄도 안전한 근거:

- 4개 DB 모두 `wal_autocheckpoint=2000`페이지가 켜져 있어 SQLite가 스스로 WAL 성장을 묶는다.
- 비정상 종료 잔재는 `InitializeDB`의 시작 대청소가 이미 회수한다.
- 주기 잡의 고유 역할(크기 임계 TRUNCATE, 밀린 회수 이어받기)은 첫 5분 안에는 필요 없다.

기대 효과: 시작 구간에서 SLOW JOB 1.4초 제거. 첫 체크포인트는 시작 5분 뒤부터.

*Orange Platform 엔드포인트 에이전트의 시작 구간 디스크 I/O 분석 리포트입니다.*
