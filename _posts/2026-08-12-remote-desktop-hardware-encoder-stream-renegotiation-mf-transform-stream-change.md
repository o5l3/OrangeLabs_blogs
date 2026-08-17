---
title: "원격제어 화질이 흐려지고 CPU가 100%로 — MF_E_TRANSFORM_STREAM_CHANGE 재협상을 놓쳐 소프트웨어 인코딩에 갇힌 사건"
excerpt: "Intel Iris Xe 노트북에서 원격제어 화질이 뭉개지고 대상 PC의 CPU가 100%로 찼다. 하드웨어 인코더가 보낸 스트림 재협상 신호(0xC00D6D61)를 로그만 찍고 넘긴 탓에 소프트웨어 폴백으로 고착된 원인 분석과, 재협상 처리·전송품질 사다리 승급 게이트 수정 기록."
category: tech
date: 2026-08-12
author: kim-tigerj
tags: [원격제어, Media Foundation, 하드웨어인코딩, H264, WebRTC, 트러블슈팅, Orange Platform]
---

## 증상

특정 노트북(Intel Iris Xe 내장 그래픽, 2560×1600)에서 원격제어 화질이 매우 나쁘고, 동시에 대상 PC의 CPU 사용률이 높았다. 작업 관리자에서 원격제어 화면 캡처 프로세스가 CPU 13.4%를 쓰고 있었다.

별개로, 회선이 아무리 좋아도 자동 전송품질이 좋은 쪽으로 올라가지 않는 문제도 확인됐다. P2P 직접 연결(UDP, RTT 13ms, 손실 0%, 같은 망)인데도 화면이 뭉개졌다.

## 원인 1 — 하드웨어 인코더 스트림 재협상 누락 (주원인)

노드에서 받은 캡처 로그의 해당 구간이다.

```
[livesource] adapter='Intel(R) Iris(R) Xe Graphics' vendor=0x8086
[livesource] capture init done 2560x1600 (GPU pipeline)
[livesource] ProcessOutput(hw) hr=0xc00d6d61
[livesource] ProcessOutput(hw) hr=0xc00d6d61
[livesource] hardware encoder stalled (no output 2.5s, frames fed=2) -> software fallback
[livesource] hardware encoder stalled -> restart in software encode
[adapt] fps down -> 20 (cpu=100%)
[adapt] fps down -> 15 (cpu=100%)
[adapt] res down -> 1802x1126 (cpu=100%)
[adapt] res down -> 1440x900 (cpu=100%)
```

### 0xC00D6D61이 무엇인가

Windows SDK `mferror.h`에 이렇게 정의돼 있다.

```
// A stream change has occurred. Output cannot be produced until the streams have been renegotiated.
#define MF_E_TRANSFORM_STREAM_CHANGE  _HRESULT_TYPEDEF_(0xC00D6D61L)
```

인코더가 "출력 형식이 바뀌었으니 다시 협상하자"고 알린 것이다. **오류가 아니라 규약이다.** 호출한 쪽이 새 출력 타입을 확정해 주기 전까지 인코더는 출력을 하나도 내지 않는다.

### 연쇄

우리 코드는 이 값을 그냥 로그만 찍고 넘어갔다(`agent/livesource.cpp`, `runHardwareLoop`의 `ProcessOutput` 처리). 그래서 이런 순서로 무너진다.

- 인코더가 재협상을 기다리며 멎는다.
- 2.5초 뒤 정지 감시가 "하드웨어 인코더가 죽었다"고 오판한다 (실제로는 우리가 응답을 안 한 것).
- 소프트웨어 인코딩으로 다시 시작한다.
- 2560×1600 소프트웨어 인코딩이라 CPU가 100%로 찬다.
- CPU 부하 조절기가 fps를 30 → 20 → 15로, 해상도를 2560×1600 → 1440×900으로 강등한다.

**화질 저하와 CPU 사용률은 같은 뿌리에서 나온 두 증상이다.**

### 언제부터인가 — 최신 버전의 회귀가 아니다

로그 전 구간을 버전별로 세었다. `0xc00d6d61`은 v0.1.0.19부터 계속 나온다. 이 노트북은 처음부터 소프트웨어 인코딩으로 돌고 있었다.

> **로그 판독 주의:** 이 파일은 7월부터 누적된 것이라 세션 배너가 1,524개였고 그중 1,493개가 v0.1.0.56 시절이다. 전체 통계를 그대로 읽으면 최근 버전의 문제로 오해한다. 최근 버전 구간만 떼어 봐야 한다.

## 원인 2 — 자동 전송품질이 정지 화면에서 영원히 못 올라간다

자동은 중상(칸 1, MaxQP 27 / 10 Mbps)에서 시작해 회선 손실을 보고 칸을 오르내린다. 칸을 올리려면 `onReceiverStats`가 손실·수신 표본을 봐야 하는데, 여기에 이런 게이트가 있었다.

```
uint32_t total = dLost + dRecv;
if (total < 50) return;   // 표본 부족 — 전환 오판 방지
```

매니저는 1.5초마다 누적 손실·수신을 보낸다. 그런데 정지 화면은 바뀌는 게 없어 초당 1프레임·0.01 Mbps밖에 안 나간다. 1.5초 동안 도착하는 패킷이 한두 개다. 매번 표본 부족으로 조기 반환하니 좋음 카운터가 단 한 번도 오르지 않는다.

회선이 좋다는 걸 증명하려면 트래픽이 필요한데, 정지 화면은 트래픽을 만들지 않는다. 그래서 영원히 증명하지 못하고 시작 칸에 갇힌다. 손실 0%, RTT 13ms, 같은 망이어도 마찬가지다. 움직임이 있어도 조건이 빡빡하다 — 표본이 찬 보고가 15초 연속 깨끗해야 한 칸 오른다. 실제 작업은 끊겼다 이어지므로 잘 채워지지 않는다.

## 수정 (remote v0.1.0.142)

| 대상 | 변경 |
|---|---|
| `agent/livesource.cpp` — 스트림 재협상 | `MF_E_TRANSFORM_STREAM_CHANGE`를 받으면 인코더가 제시하는 출력 타입 목록에서 H.264를 골라 우리 비트레이트를 얹어 `SetOutputType`으로 다시 확정하고 하드웨어 인코딩을 유지한다. 새 타입의 첫 프레임은 IDR로 보내 뷰어가 바로 그리게 한다. 재협상이 실패하면 기존 동작(정지 감시 → 소프트웨어 폴백)이 그대로 살아 있어 지금보다 나빠지지 않는다 |
| `agent/livesource.cpp` — 사다리 승급 | 표본이 50개 미만이어도 잃은 게 0이고 받은 게 있으면 좋음으로 센다. 잃은 게 있으면 표본이 적어도 나쁨. 아무것도 안 왔을 때만 판단을 보류한다. 승급 간격(10회 ≈ 15초)은 그대로라 조급하지 않고, 회선이 실제로 나쁘면 손실이 잡히는 순간 2회(≈3초)만에 내려온다 |
| `agent/livesource.cpp` — GOP | 1초(= fps) → 약 10초. 키프레임은 새 뷰어 접속·큐 넘침 복구에서 이미 필요할 때 보내고 있어 주기적 키프레임은 예산만 먹는다. 다만 GOP는 초가 아니라 프레임 단위라 정지 화면에서는 잘 걸리지 않는다 — 움직임이 많을 때만 효과가 있는 변경이다 |

## 검증

대상 PC의 CPU 사용률이 확연히 떨어진 것을 확인했다(v0.1.0.142, 문제의 Intel Iris Xe 노트북). 하드웨어 인코딩이 유지되고 있다는 뜻이다. 로그에서는 아래가 확인되어야 한다.

- `[livesource] output stream change -> renegotiated (staying on hardware)`가 찍혀야 한다.
- `hardware encoder stalled ... -> software fallback`이 없어야 한다.
- `[adapt] res down` / `[adapt] fps down`이 없어야 한다.
- 뷰어 상단 해상도가 2560×1600을 유지해야 한다.
- 자동 전송품질은 정지 화면에서 약 15초 뒤 `[ladder] up -> 0`이 찍혀야 한다.

## 진단 과정에서 기각한 가설 (기록)

- **최신 exe의 회귀다** — 기각. `0xc00d6d61`은 v0.1.0.19부터 나온다. 이 기계는 처음부터 소프트웨어 인코딩이었다.
- **코덱 설정을 출력 타입 확정 전으로 옮기면서 1초 GOP가 비로소 먹기 시작해 화질이 떨어졌다** — 기각. 설정이 실제로 반영되기 시작한 것은 맞지만, 이 노트북의 화질 저하는 소프트웨어 폴백 뒤의 해상도·fps 강등으로 전부 설명된다. GOP는 프레임 단위라 영향이 훨씬 작다.
- **정지 화면에서 fps가 1~2로 나오는 것이 문제다** — 기각. 바뀌는 것이 없으면 안 보내는 설계(send-on-change)대로다. 정상이다.

*Orange Platform 원격제어 하드웨어 인코더 스트림 재협상 트러블슈팅 리포트입니다.*
