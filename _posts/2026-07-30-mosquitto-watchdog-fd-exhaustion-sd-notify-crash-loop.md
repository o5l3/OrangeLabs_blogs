---
title: "mosquitto 2.1 watchdog 오탐 crash loop — FD 1024 고갈로 sd_notify가 조용히 유실될 때"
excerpt: "1,000대 동시 접속 부하 테스트 중 mosquitto broker가 8분 주기로 SIGABRT 강제 종료를 반복하던 crash loop를 추적한 기록. systemd 기본 FD soft 한도(1024) 고갈 → sd_notify()용 소켓 생성 실패 → watchdog ping 유실 → systemd가 멀쩡한 broker를 hang으로 오판해 사살하던 인과 체인을 규명하고, LimitNOFILE=65536 drop-in으로 해결했다."
category: tech
date: 2026-07-30
author: smahn9123
tags: [mosquitto, MQTT, systemd, watchdog, sd_notify, 파일 디스크립터, 트러블슈팅, Orange Platform]
---

## 개요

가상 agent 부하 테스트(~1,000대 동시 접속) 중 테스트 서버(192.168.0.207)에서 mosquitto broker가 **8분 주기로 SIGABRT 강제 종료를 반복**하는 crash loop가 발견됐다. 원인은 mosquitto 2.1 벤더 systemd 유닛의 FD 한도(soft 1024) 고갈이며, systemd drop-in에 `LimitNOFILE=65536`을 추가하여 해결했다.

## 증상 / 타임라인 (2026-07-30)

| 시각 | 사건 |
|---|---|
| 13:46:02 | 가상 agent ~1,000대가 3182(TLS)로 동시 접속 시작. 1분 내 1,072대 CONNECT 완료 (broker 정상 처리) |
| 13:46:51 | FD 1024 도달 → `Unable to accept new connection, system socket count has been exceeded` 발생 (총 2,497회). 신규 접속 거부 시작 |
| 13:49:24 | systemd watchdog timeout (3min) → 멀쩡히 동작 중인 broker SIGABRT 사살 → 자동 재시작 → 재접속 폭풍 |
| 15:01, 15:09 | agent 재기동 시마다 동일 사이클 반복 (재시작 → FD 고갈 → 3분 뒤 사살) |
| 15:11 | `LimitNOFILE=65536` drop-in 적용 → crash loop 종료, EMFILE 0건 확인 |

## 원인 분석 (인과 체인)

1. **mosquitto 2.1 벤더 유닛의 신설 watchdog**: 2.0에 없던 `Type=notify` + `WatchdogSec=3min`이 2.1 패키징에 추가됐다. broker는 메인 루프 매회 `watchdog__check()`를 호출해 90초마다 `sd_notify(0, "WATCHDOG=1")`를 전송한다 (`src/watchdog.c`).

2. **FD 한도 미조정**: 벤더 유닛이 `LimitNOFILE`을 설정하지 않아 systemd 기본 soft 1024로 동작한다. 접속 1개 = FD 1개이므로 실질 수용량은 약 1,000 접속.

3. **sd_notify의 숨은 FD 의존**: `sd_notify()`는 호출할 때마다 새 유닉스 소켓을 `socket()`으로 생성한다 (libsystemd `sd-daemon.c`). FD 고갈 상태에서는 EMFILE로 실패하는데, mosquitto는 반환값을 확인하지 않고 `next_ping`을 갱신한다 → watchdog ping이 조용히 유실된다.

4. **오탐 사살**: systemd는 3분간 ping 부재를 hang으로 판정해 SIGABRT를 보낸다. 로그상 broker는 사살 직전까지 분당 200~250건의 접속/해제를 정상 처리 중이었다 (hang 아님).

**핵심**: watchdog kill은 증상이고, 근본 원인은 FD 1024 고갈이다. 신규 접속 거부("3182 안 됨")도 같은 원인의 다른 증상이다.

## 조치

- systemd drop-in(`/etc/systemd/system/mosquitto.service.d/docker-dependency.conf`)에 `LimitNOFILE=65536` 추가.
- 설치/업데이트 스크립트(`install/setup.sh`, `install/update.sh`)의 drop-in 생성 heredoc에 반영 (1:1 동기).
- 운영 문서 "mosquitto 2.1 함정 목록"에 기록.
- **watchdog은 유지** — FD가 충분하면 ping이 정상 발송되므로 본래 목적(진짜 hang 감지 + 자동 복구)대로 유용하다. 무력화(`WatchdogSec=0`)는 증상 은폐라서 배제했다.

## LimitNOFILE=65536 산정 근거

- **한도는 예약이 아니라 상한**: 올려도 사전 할당 자원은 없다. 실제 메모리는 접속 수에 비례하며 별도로 `memory_limit 1GB`(mosquitto.conf)가 상한이다. Linux `epoll` 사용이라 `select()`의 1024 제약도 무관하다.
- **최악 순간 여유**: 제품 스펙 최대 10,000 agent 기준, broker 재시작/네트워크 순단 후 10,000대 동시 재접속 시 구 소켓(half-open/close 대기)과 신규 소켓이 공존해 순간 FD가 평시의 1.5~2배(20,000~25,000)까지 상승할 수 있다. 여기에 브라우저 WebSocket(위젯당 1개), 리스너 4개, 내부 FD가 가산된다.
- **생태계 관행 값**: MongoDB 공식 유닛 `LimitNOFILE=64000`, redis 권장 65535와 동일한 급이다.

## 리뷰 포인트 / 후속 검토

- 고객사 노드 1,000대 초과 시 동일하게 재발하는 지뢰였다 — 기존 설치 서버는 `update.sh` 재실행으로 drop-in이 갱신되어야 적용된다.
- (선택) 제품 스펙 10,000대를 강제하려면 FD 한도가 아니라 mosquitto.conf `max_connections`(예: 12000)가 올바른 수단이다 — 초과분이 EMFILE 대신 MQTT 레벨에서 우아하게 거부되고 sd_notify용 FD가 상시 확보된다. 채택 여부 검토 필요.
- 업스트림 mosquitto 2.1.3 ChangeLog에 관련 수정은 없다 (sd_notify 반환값 미확인은 잠재 업스트림 리포트 대상).

*Orange Platform 운영 중 발견한 mosquitto watchdog crash loop 트러블슈팅 리포트입니다.*
