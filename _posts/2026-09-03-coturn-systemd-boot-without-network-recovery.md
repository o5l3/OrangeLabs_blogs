---
title: "네트워크 없이 부팅하면 coturn이 살아나지 않는다 — systemd 재시도 한계와 도커 브리지 함정"
excerpt: "현장 설치에서 네트워크를 연결하지 않은 채 서버를 먼저 부팅하면 coturn이 기동에 실패한 뒤 네트워크를 꽂아도 스스로 올라오지 않았다. 원인은 세 가지 — 리슨 주소가 루프백뿐이면 종료, systemd 기본 시작 제한(10초 내 5회)에 걸려 영구 포기, 그리고 나중에 뜬 도커 브리지에만 바인딩해 active인데 닿지 않는 상태. 세 원인을 컨테이너로 재현하고 드롭인 하나로 전부 막은 과정."
category: tech
date: 2026-09-03
author: smahn9123
tags: [systemd, coturn, TURN, Docker, 네트워크, 부팅순서, 트러블슈팅, Orange Platform]
---

## 배경

현장 담당자가 2026-09-03 보고한 문제다. 업체 방문 설치 시 IP 충돌·네트워크 문제에 대비해 네트워크를 연결하지 않은 채 서버를 먼저 부팅하는 경우가 있는데, 이때 부팅 로그에 `Failed to start coturn.service - coTURN STUN/TURN Server`가 반복해 뜬다. 현장 제안은 `coturn.service`에 `After=network-online.target` / `Wants=network-online.target`을 넣어 달라는 것이었다.

조사 결과 현장 제안만 넣거나 재시도만 넣으면 「보이는 실패」가 「안 보이는 실패」로 바뀐다는 것을 확인했다. 아래 2절 ③이 이번 조사의 핵심이다.

## 1. 영향

로그 소음보다 뒤에 남는 상태가 문제다.

- coturn이 기동에 실패하면 네트워크가 연결된 뒤에도 스스로 올라오지 않는다. 사람이 `systemctl restart coturn` 하기 전까지 그 서버는 원격제어 릴레이가 꺼진 채로 운영된다.
- 매니저·에이전트는 정상 동작하므로 겉으로 드러나지 않는다. 원격제어를 실제로 시도할 때 화면이 뜨지 않는 것으로 처음 드러난다.
- 대상: coturn이 들어간 패키지로 설치·업데이트한 서버 전부. 그 이전 설치분에는 coturn 자체가 없어 해당 없음.

## 2. 원인 — 세 가지가 겹친다 (전부 컨테이너로 재현)

### ① 주소 없이 뜨면 죽는다

`conf.d/turnserver.conf`는 `listening-ip`를 지정하지 않는다. 전 인터페이스 바인딩이 의도이고 외부 접근은 UFW 3184/tcp·3184/udp로만 통제한다. 이 경우 coturn은 기동 시점에 인터페이스를 훑어 리슨 주소를 스스로 정하는데, 루프백도 후보로 잡지만 루프백만 있으면 거부하고 종료한다.

```
0: : NO EXPLICIT LISTENER ADDRESS(ES) ARE CONFIGURED
0: : ===========Discovering listener addresses: =========
0: : Listener address to use: 127.0.0.1
0: : Listener address to use: ::1
0: : ERROR: main: Cannot configure any meaningful IP listener address
(exit 255)
```

### ② 0.1초 간격 5회 만에 영구 포기한다

Ubuntu coturn 4.6.1-1build4 벤더 유닛은 이미 `Restart=on-failure`를 갖고 있다. 다만 `RestartSec`가 없어 기본 100ms로 돌아 systemd 기본 시작 제한(10초 내 5회)에 즉시 걸린다.

```
coturn.service: Scheduled restart job, restart counter is at 5.
coturn.service: Start request repeated too quickly.
coturn.service: Failed with result 'exit-code'.
Failed to start coturn.service - coTURN STUN/TURN Server.
```

현장이 본 「에러가 계속 발생」이 이 5회 반복이다. 그리고 시작 제한에 걸린 뒤에는 systemd가 재시도를 포기하므로, 네트워크가 나중에 연결돼도 아무 일도 일어나지 않는다.

### ③ 도커 브리지만 있어도 기동에 성공한다

coturn은 「루프백이 아닌 주소」면 무엇이든 받아들인다. 도커 브리지도 포함된다. 실서버에는 docker0(172.17.0.1)와 orange_net 브리지(172.30.0.1)가 있고, 이 둘은 랜선과 무관하게 존재한다. 브리지만 있는 상태에서 띄운 결과:

```
$ systemctl is-active coturn
active
$ ss -tln | grep 3184
127.0.0.1:3184   172.17.0.1:3184   172.30.0.1:3184   [::1]:3184
```

active인데 에이전트도 매니저도 닿을 수 없는 주소에만 붙어 있다. coturn은 기동 후 인터페이스를 다시 훑지 않으므로 나중에 랜선을 꽂아도 그 NIC에는 영영 붙지 않는다.

실서버 부팅 순서가 정확히 이 함정을 만든다. coturn은 `After=network.target`이라 이르게 뜨고 docker는 늦게 뜬다. ②만 고쳐 재시도 간격을 늘리면 이렇게 된다.

```
t+2s    coturn 기동 → 루프백뿐 → 실패
t+20s   docker 기동 → docker0 / orange_net 브리지 생성
t+30s   coturn 재시도 → 브리지 주소 발견 → active (브리지에만 바인딩)
t+5m    랜선 연결 → coturn 은 그대로. 재탐색 없음
결과    systemctl is-active 는 active, 원격제어는 안 됨
```

지금은 systemd가 영구 포기해서 이 상태에 빠지지 않는다. 즉 순진한 수정은 현재보다 나쁘다.

우리 스택에서 coturn만 이렇다. mongod·redis·mosquitto는 도커 브리지 주소로 명시 바인딩하고 nginx는 와일드카드라, 네트워크 없는 부팅에서도 무관하다. coturn만 `listening-ip`가 없어 기동 시 인터페이스를 탐색하고 루프백뿐이면 종료한다.

## 3. 현장 제안 검토

`After`/`Wants=network-online.target`은 맞는 방향이다. 「기동 시점에 주소가 필요한 서비스」가 써야 할 대기 지시자가 이것이고, 정상 부팅에서 실패 자체를 없앤다. 다만 이것만으로는 두 가지가 남는다.

- 늦게 연결되면 결과가 같다. `network-online.target`은 `systemd-networkd-wait-online`이 성공하든 타임아웃(기본 120초)으로 실패하든 도달한다. 2분 안에 꽂으면 해결되고 그 뒤에 꽂으면 그대로 실패한다.
- 실패 후 복구 경로가 없다. 2절 ②가 그대로 남는다.

부팅 시간은 늘지 않는다. docker.io 패키지의 유닛이 이미 같은 타깃을 `Wants`로 걸고 있어 `wait-online`은 어차피 매 부팅 돈다(패키지에서 직접 확인).

## 4. 적용

벤더 유닛을 편집하지 않고 드롭인으로 넣는다. `/etc` 아래라 apt 업그레이드가 건드리지 않는다.

```ini
# /etc/systemd/system/coturn.service.d/orange.conf
[Unit]
After=network-online.target
Wants=network-online.target
StartLimitIntervalSec=0

[Service]
ExecStartPre=/bin/sh -c 'ip -4 -o addr show scope global | grep -qvE "^[0-9]+: (docker|br-|veth)"'
Restart=on-failure
RestartSec=30
```

| 항목 | 맡는 문제 |
|------|-----------|
| After / Wants=network-online.target | ①. 정상 부팅에서 실패 자체를 없앤다. 현장 제안 그대로 |
| RestartSec=30 + StartLimitIntervalSec=0 | ②. 0.1초 폭주와 영구 포기를 없애고, 네트워크가 언제 연결되든 스스로 붙는다 |
| ExecStartPre 주소 검사 | ③. 도커 브리지를 제외한 글로벌 IPv4가 있을 때만 통과. 미달이면 비0 종료라 잘못된 바인딩 없이 재시도 대기로 간다 |

`Restart=on-failure`는 벤더 값과 같지만 의도를 드러내려 드롭인에 다시 적는다. `StartLimitIntervalSec=0`은 `RestartSec=30`이면 산술적으로 시작 제한에 걸리지 않아 없어도 되지만, 그 계산에 기대지 않으려고 명시한다. 주소 검사에서 제외하는 이름은 `docker*`·`br-*`·`veth*`이고, 운영자가 만든 `br0` 같은 이름은 정상 통과한다.

받아들이는 부작용: 네트워크가 끝내 없는 장비는 30초마다 실패 로그가 남는다(journald `SystemMaxUse=2G` 상한 안에서 순환). 설정 파일이 잘못돼 기동하지 못하는 경우도 같은 경로로 반복 로그가 된다. 두 경우 모두 `systemctl status coturn`에 사유가 그대로 보인다.

검토했다가 택하지 않은 안: `listening-ip`를 서버 IP로 명시. 멀티홈 구성을 깨뜨리고 IP 변경 때 고쳐야 할 파일이 하나 더 늘어난다. 현재 템플릿 주석이 명시적으로 배제한 방식이다.

## 5. 검증 결과

환경: ubuntu:24.04 컨테이너, coturn 4.6.1-1build4(실서버와 동일 버전), systemd를 PID 1로 기동, `--network none`으로 랜선 미연결 재현. 랜선 연결과 도커 브리지는 dummy 인터페이스로 재현. 드롭인은 손으로 쓴 것이 아니라 커밋된 설치 스크립트가 실제로 출력한 파일을 그대로 썼다.

| 확인 항목 | 결과 |
|-----------|------|
| 루프백만 있을 때 | Cannot configure any meaningful IP listener address, exit 255 |
| 벤더 유닛 그대로일 때 | 5회 재시도 후 Start request repeated too quickly, 영구 실패 |
| 도커 브리지만 있을 때 (수정 전) | active이면서 172.17.0.1·172.30.0.1에만 바인딩 |
| 드롭인 적용 + 랜선 없음 | activating (재시도 대기) |
| 드롭인 적용 + 도커 브리지만 | activating 유지. 3184 리슨 0개 — 잘못된 바인딩 없음 |
| 드롭인 적용 + 랜선 연결 | 30초 뒤 자동으로 active, 실제 NIC 192.168.55.10:3184에 바인딩 |
| 이미 시작 제한에 걸려 멈춘 유닛 | 드롭인 배치 + daemon-reload + start로 복구됨. reset-failed 불필요 |

마지막에서 두 번째 항목이 중요하다. 이미 고장 난 서버도 업데이트 한 번이면 복구된다. 따로 손봐야 할 서버 목록이 없다.

실장비 부팅 검증은 다음 배포 때 두 가지로 본다. 정상 네트워크 서버 재부팅에서 기존 동작 변화가 없는지, 그리고 랜선 미연결 부팅 후 나중에 연결했을 때 30초 안에 자동 복구되어 `ss -tln | grep 3184`에 실제 서버 IP가 나오는지.

*Orange Platform 서버 패키지의 coturn 부팅 실패 분석 리포트입니다.*
