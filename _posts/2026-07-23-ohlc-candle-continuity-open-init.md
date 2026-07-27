---
title: "부하 지표 캔들(OHLC)의 봉이 이어지지 않는 이유 — 시가를 직전 종가로 잇도록 버킷 전환 수정"
excerpt: "노드 부하를 OHLC 캔들로 시각화하던 중, 봉과 봉의 시가가 이어지지 않고 튀는 결함을 발견한 기록. 5분 버킷 전환 시 open을 새 버킷의 첫 측정값으로 잡던 것을 직전 버킷의 종가로 잇도록 바꾸고, high/low 초기화 기준을 새 버킷 open으로 정정했다."
category: tech
date: 2026-07-23
author: kim-tigerj
tags: [OHLC, 캔들 차트, 시계열 집계, 에이전트, 성능 지표, C++, Orange Platform]
---

## 배경

노드 위젯의 Live Agent 부하 캔들을 정리하던 중 발견했다. 캔들 윗꼬리가 유난히 길어 조사한 결과, 윗꼬리 자체는 실제 순간 스파이크로 확인됐으나(집계 정상, mean 10~16% 정상 범위·high 45~75% 실측) **봉과 봉이 이어지지 않는 별도의 결함**이 드러났다.

## 결함

`CSummaryData.h`의 `NextSummary`(5분 버킷 전환 시).

- `bSet=true`(CPU/Memory/IO/Handle 전부)일 때 `open`을 **새 버킷의 첫 측정값**으로 잡는다. 캔들 시가는 직전 버킷 종가를 이어야 하는데 끊긴다 — 봉마다 시가가 튄다.
- `high`/`low` 초기화가 새 버킷 기준이 아니라 직전 버킷의 마지막 `close` 기준이다.

## 수정안

```cpp
ptr->open = ptr->close;            //  bSet 무관, 직전 종가를 새 시가로
if (bSet) ptr->close = value;
else      ptr->close += value;
ptr->high = ptr->open;             //  open 에서 출발 (몸통이 심지를 벗어나지 않게)
ptr->low  = ptr->open;
if (ptr->close > ptr->high) ptr->high = ptr->close;
if (ptr->close < ptr->low)  ptr->low  = ptr->close;
```

- `SetSummary`는 정상이므로 손대지 않는다.
- `bSet=false`(간헐 start/stop) 경로도 동일 방향이라 무해하다(교차검증 확인).
- `total`/`mean` 산술평균은 정상이다.

과거 버킷 데이터는 그대로 남는다. 빌드·재시작 후 새로 쌓이는 버킷부터 봉이 이어진다.

---

*이 글은 Orange Platform 에이전트의 부하 지표 시계열 집계 분석 및 트러블슈팅 리포트입니다.*
