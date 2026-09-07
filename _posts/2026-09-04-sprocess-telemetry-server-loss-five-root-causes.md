---
title: "상주 프로세스 텔레메트리가 서버에서 간헐 누락되는 이유 — 에이전트 4건·서버 1건 복합 원인 전수 분석"
excerpt: "상주 프로세스(Meter.exe)의 sprocess가 서버 raw·hourly·daily에 간헐적으로 누락되는 문제를 에이전트·서버 코드와 실측으로 전수 분석했다. 원인은 단일이 아니라 발행 throttle, 열린 버킷의 timestamp NULL, QoS 0 전송 유실, 정렬·처리량 상한에 따른 과거 band starvation, 서버의 지각 도착 hour 미재집계까지 다섯 가지의 복합이었다. '변화 없으면 hourly 누락'이라는 의심 경로는 아님으로 확인한 과정을 포함한다."
category: tech
date: 2026-09-04
author: kim-tigerj
tags: [텔레메트리, MQTT, QoS, SQLite, 집계, 데이터파이프라인, 트러블슈팅, Orange Platform]
---

## 개요

상주 프로세스(`Meter.exe` 등)의 sprocess가 서버 `sprocess`(raw)·`sprocess_hourly`·`sprocess_daily`에 간헐적으로 누락되는 문제를 agent·server 코드와 실측 데이터로 전수 분석했다.

원인은 단일이 아니라 에이전트 4건 + 서버 1건의 복합이며, 일부는 수정 완료다. "ProcessStat2에서 변화가 없으면 hourly를 누락시킨다"는 의심 경로는 아님으로 확인했다. 모든 항목은 코드·SQL·실측으로 근거를 제시하며, 미확정은 별도 표기한다.

## 원인 요약

| 구분 | 위치 | 원인 | 상태 |
|------|------|------|------|
| A | agent | CSProcessCache 1시간 throttle → 발행 자체 누락 | 수정 완료 |
| B | agent | Create-only 버킷 timestamp NULL → 업로드 SELECT 제외 | 수정 완료 |
| C | agent | QoS 0 + 발행 전 마킹 → 전송 유실, 재전송 없음 | 미수정 |
| D | agent | DESC + LIMIT 처리량 상한 → 과거 band starvation | 판단 대기 |
| E | server | aggregator가 지각 도착 hour를 재집계 안 함 | 미수정 |

## 확인된 현상 (실측)

agent 버킷 hour와 server hour는 정확히 1시간 어긋나 저장된다(server hour = agent 버킷 hour + 1). 버킷 고유값 SystemTime으로 1:1 대조:

```
agent hour(UTC)   SystemTime       -> server hour   서버 존재
03:00             17143.875           04:00          O
04:00             86320.5             05:00          O
  ... (연속) ...                                     O
12:00             81029.015625        13:00          O
13:00             95143.5             14:00          X   <-- 누락
14:00             84897.375           15:00          O
  ... (연속) ...                                     O
22:00             82387.78125         23:00          O
```

모든 SystemTime 일치 → shift = +1시간 확정. agent 13:00(UTC) 버킷만 서버에 없음(UploadCount=1, LastUploadTime=2026-07-13T13:42Z). 에이전트는 전송 마킹했으나 서버 미기록 = 원인 C(전송 유실) 사례.

## 원인 A — 에이전트 발행 누락 (CSProcessCache 1시간 throttle) · 수정 완료

`UploadSPROCESSByPolicy`는 매 사이클 pending 버킷을 SELECT → 각 SUID의 부모 family를 `GetSPROCESS(SUID, slist, flist)`로 채우고 → row의 SUID가 slist에 있을 때만 발행한다.

`CAgentHelper3.Detect.cpp`의 `GetSPROCESS`가 `CSProcessCache`(SUID 키, 1시간 TTL)를 써서, 한 SUID가 한 번 처리되면 1시간 동안 그 SUID의 후속 시간버킷을 slist에 안 채웠다. 결과: 사이클당 SELECT 100개 중 실제 발행 약 10~15개(약 85% skip), pending이 약 4,000개까지 폭증, Meter 같은 상주 프로세스의 최근 데이터가 서버로 안 감(debug.log에 다른 프로세스 `SPROCESS(1)`은 찍히는데 Meter만 빠지는 증상). 서버가 못 받은 게 아니라 에이전트가 조건부로 발행을 누락한 것.

**수정 (반영 완료)**

- `CAgentHelper3.Detect.cpp` — `GetSPROCESS`에서 `CSProcessCache` 제거, 항상 family 채움(버킷별 중복은 SELECT의 UploadTimestamp 조건이 담당).
- `CAgentHelper3.h` — `CSProcessCache` 상속 제거.
- `db/summary.json` — pending SELECT 2개: LIMIT 100 → 512, 재선정 TTL 1800000(30분) → 300000(5분), ORDER BY hour ASC → DESC.

효과(실측): 재시작 후 사이클이 512건 발행 + `SPROCESS(1) …Meter.exe`를 찍고, Meter 최신 시각이 서버 raw·hourly에 현재 시간까지 도달.

## 원인 B — Create-only 버킷 timestamp NULL → 업로드 제외 · 수정 완료

"변화 없음"이 아니라 hourly 버킷의 timestamp 생성 시점이 문제였다.

| SQL | timestamp 설정 | 시점 |
|-----|----------------|------|
| SPROCESS_HOURLY_CREATE (INSERT) | (기존) 안 함 → NULL | 버킷 시작(hour 첫 이벤트) |
| SPROCESS_HOURLY_UPDATE | `timestamp=strftime('%s','now')*1000` | 버킷 마감(Flush)·종료 이벤트 |
| 업로드 SELECT WHERE | `h.timestamp IS NOT NULL` 요구 | — |

Create만 되고 UPDATE(Flush) 안 된 hourly 행은 timestamp가 NULL이라 업로드 대상에서 제외된다.

언제 발생하나:

- 재시작/크래시가 그 hour 버킷이 열려 있는 중 발생(가장 흔함). 버킷은 메모리(m_data)에 있다가 재시작으로 사라지고 DB엔 timestamp NULL인 Create 행만 남는다. 재시작 후 Rotate는 이전 bucketTime이 0이라 옛 버킷을 Flush하지 않음(`CSProcessHourly.h:708`) → 그 hour는 영영 timestamp NULL → 업로드 안 됨 → 서버 구멍. 개발 PC는 리빌드·테스트 재시작이 잦아 이 구멍이 자주 생긴다.
- 운영에서도 현재 열린 hour는 마감(다음 hour 첫 이벤트)까지 업로드 불가 → 최신 hour는 늘 한 텀 지연.

**수정 (반영 완료)**: `SPROCESS_HOURLY_CREATE`의 INSERT에 timestamp 컬럼 + `strftime('%s','now')*1000` 리터럴 추가. 열린 hour도 생성 즉시 업로드 후보가 되고, 마감 UPDATE가 timestamp를 갱신해 재업로드 게이트(`h.timestamp > h.UploadTimestamp + 300000`)를 통과, 최종값으로 덮어쓴다(서버 `$merge replace` 안전). `INSERT OR IGNORE`라 이미 있는 행은 no-op. timestamp 컬럼이 이미 있어 스키마 변경(ALTER)·마이그레이션은 불필요하다.

잔여(이 수정의 결함 아님): ① 최신 hour는 마감 전까지 부분값(첫 측정 기준 CPU/pscore)으로 보일 수 있음(마감 때 교정). ② 재시작으로 메모리 버킷이 소실된 옛 hour는 최종 Flush가 안 와 부분값으로 남음 → 100% 메우려면 시작 시 고아(Create-only) 행 보강 flush 별건 필요.

## 원인 C — 전송 유실 (QoS 0 + 재전송 없음) · 미수정

`CAgentHelper3.UploadSPROCESSByPolicy.cpp`는 `:118`에서 먼저 "업로드됨" 마킹(UploadTimestamp/UploadCount)을 하고 → `:124`에서 그 뒤 QoS 0(`CONN_MQTT_QOS0`, fire-and-forget)로 발행한다. QoS 0는 전달 보장이 없어 broker/서버 ingest에서 drop되면 서버 미수신. 재선정 조건이 timestamp 기반이라 이미 마킹된 완성 버킷은 재대상이 안 됨 → 영구 유실.

미확정: drop 지점(broker vs 서버 ingest)은 미측정. 수정 방향은 QoS 1 상향, 또는 발행 성공 확인 후 마킹, 또는 미수신 재전송 메커니즘.

## 원인 D — DESC + LIMIT 처리량 상한 → 과거 band starvation · 판단 대기

원인 A 수정에서 정렬을 hour DESC(최신 우선) + LIMIT 512 + 5분 TTL 재업로드로 바꾼 뒤, 매 사이클 pending이 512 근처로 차 있으면 가장 오래된 버킷이 매번 잘려 발행되지 못한다. 실측(SUID 281476647968923): 07-14 16·18~23Z 등 9개 버킷이 미전송으로 잔존. ASC→DESC 전환 시점이 겹쳐 중간 band가 고아가 됐다.

판단 대기: 과거 band를 메꾸려면 (a) 재업로드 TTL 상향, (b) LIMIT 상향으로 pending<LIMIT 유지(순서 무관 전량 발행), (c) 최신·과거 혼합 정렬 중 택한다.

## 원인 E — 서버 지각 도착분 미재집계 · 미수정

`process_aggregator.py`의 hourly 집계 트리거는 둘뿐이다: `:356` cron(매시 :05)=직전 hour만, `:363` partial(:15~:55)=현재 hour만. 지나간 hour를 다시 집계하는 코드가 없어, 어떤 hour의 raw가 (그 hour+1):05 이후 도착하면 `sprocess_hourly`에 영영 미반영된다(raw엔 존재해도). `sprocess_daily`는 현재 day 전체를 매시 재집계해 당일 지각분은 자가보정되나, 유실된 raw(원인 C)와 과거 day 지각분은 미반영이다.

실측 재확인(SUID 281476647968923): raw 12 hour 중 hourly엔 8 hour만, 4개(07-14 12·14·15Z, 07-15 01Z)가 raw엔 있으나 hourly 미생성.

수정 방향: hourly cron이 직전 1시간이 아니라 직전 N시간(예: 3~4) 재집계. `$merge whenMatched:"replace"`가 멱등이라 반복 집계는 안전하다.

## 아님으로 확인 — "변화 없음 → hourly 누락"

`ProcessStat2`의 `AddCounters`가 bChanged를 계산하지만 hourly 누적은 무조건 수행된다. bChanged는 var.changed 플래그만 세우고, 그건 `Stat2.cpp:162`의 실시간 `ProcessStat1 Notify`만 게이트한다. hourly 누적 `OnStat`(`CSProcessHourly.h:405`)은 변화 여부와 무관하게 항상 누적 + 버킷 Create/Flush. 즉 no-change로 hourly가 빠지지 않는다(실측: orange.exe 58/58 hour, 누락 0). 교차검증도 이 결론을 확정했다.

## 스키마 참고

- `sprocess_hourly`·`sprocess_daily`는 node_id 필드가 없고 SUID로만 키된다(SUID가 기기독립 역할 식별자 → 역할 단위 집계). node_id로 조회하면 0건이 나오므로 SUID로 대조해야 한다.
- 분석 대상 Meter SUID가 281476703477915 → 281476647968923로 바뀐 것은, SUID가 부모 역할(PRunUID=ProcName|Product|Company|Signer 해시)을 포함하는데 개발 중 무서명 Debug orange.exe 리빌드로 부모 Signer가 비면서 자식 SUID가 재산정된 결과다. 서명이 고정되는 프로덕션에선 안정적이며 본 무결성 문제와는 별개다.

*Orange Platform 프로세스 텔레메트리 파이프라인 누락 분석 리포트입니다.*
