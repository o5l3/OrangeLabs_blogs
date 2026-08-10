---
title: "WebRTC 기반 자체 원격 화면 제어 구축기 — DXGI 캡처·MF H.264·libdatachannel, 보안 데스크톱과 TCP 릴레이까지"
excerpt: "MeshCentral·RustDesk를 그대로 쓰지 않고 DXGI+Media Foundation H.264+libdatachannel로 웹 뷰어 원격 제어를 자체 구현하고, 로그인 전 보안 데스크톱 제어·세션 전환 무이음·UDP 전면 차단망 TCP both-relay까지 완성한 기록."
category: tech
date: 2026-08-06
author: kim-tigerj
tags: [WebRTC, 원격제어, DXGI, MediaFoundation, H264, coturn, TURN, ICE, Windows에이전트, Orange Platform]
---

## 개요

상용/오픈소스 원격제어(MeshCentral·RustDesk)를 그대로 채택하지 않고, 오픈소스를 참고해 Orange 전용 원격 화면 제어를 자체 개발한다. 목표는 **PC(에이전트) → Server → Manager(Web UI)** 로 연결되는 WebRTC 기반(가능하면 P2P) 구조다.

**현재 상태** — 설계·구현을 넘어 로그인 안 된 보안 데스크톱까지 재연결 없이 제어, 세션 전환 견고성, 제품 통합(MQTT 시그널링·manager-web 위젯), 그리고 UDP 전면 차단 기업망(다중 NAT VM 포함)에서 양방향 TCP 릴레이(both-relay) 종단 동작을 완성했다. 자체 C++ 에이전트(`orange.remote.exe`)가 DXGI로 화면을 캡처하고 Media Foundation H.264로 인코딩해 libdatachannel WebRTC 미디어 트랙으로 브라우저 `<video>`에 실시간 스트리밍한다. 입력 주입·별도 커서 채널·적응형 비트레이트·자동 해상도·자체 STUN/TURN·서비스 상주, 그리고 로그인/잠금/UAC 보안 데스크톱까지 무이음으로 잡는 2프로세스 분리구조까지 구현·검증했다. 세계 순위권 제품(Chrome Remote Desktop·Parsec·AnyDesk·RustDesk)과 Windows 내부동작을 리서치로 교차검증 — 우리 분리구조가 Chrome Remote Desktop과 동일한 정답 토폴로지임을 확인했다.

## 왜 자체 개발인가

- **MeshCentral** — 웹 뷰어는 적합하나 화질 나쁨·피제어 CPU 부하 큼(JPEG 타일 방식).
- **RustDesk** — 화질/CPU는 월등하나 (1) 제어측이 네이티브 앱이라 웹 뷰어 부적합 (2) AGPL 라이선스 부담.

둘 다 오픈소스이므로 소스를 참고해, 우리 입맛에 맞는 기능만 넣어 개발 편한 **C/C++**로 만든다.

## 목표 아키텍처

```
PC(에이전트, C++)                Server                        Manager(Web UI)
─────────────────               ──────                        ───────────────
DXGI 캡처 + MF H.264       ──▶  시그널링(Orange MQTT 채널)   ──▶  브라우저 WebRTC
 (HW MFT, SW 폴백)              STUN/TURN(coturn)                  <video> 렌더 + 입력
      └────────── WebRTC 미디어 트랙(SRTP), 직접 P2P 시도 → 실패 시 coturn 릴레이 ──────┘
```

- 시그널링은 Orange MQTT 채널로 통합 완료 — yagent가 `set.REMOTE` 명령으로 remote를 Session 0에서 실행·수명관리(초기 개발엔 WebSocket 사용, 폐기됨).
- 입력(마우스/키보드)은 같은 WebRTC 연결의 데이터 채널 역방향.

### WebRTC 채택 근거

맨땅 구현이 아니라, WebRTC가 하드 문제를 다 넘겨받는다:

- **NAT 통과·P2P** — ICE/STUN/TURN 내장.
- **브라우저가 네이티브 뷰어** — WebRTC로 받은 H.264를 브라우저가 하드웨어 디코딩. 웹 디코더 불필요(웹 뷰어 요건 해결).
- **암호화(SRTP)·혼잡제어·전송·지터버퍼 내장** → 폐쇄망·릴레이(회선 흔들림)에 유리.

→ 화질(하드웨어 코덱)과 웹 뷰어를 동시에 충족. Mesh·Rust가 각각 하나씩 놓친 걸 한 번에.

## 재사용 vs 자체 개발 경계

원칙: 이미 맞는 게 있으면 새로 구현하지 않는다.

| 구분 | 항목 | 채택 |
|------|------|------|
| 재사용 | 시그널링(SDP/ICE 교환) | Orange MQTT 채널 |
| 재사용 | STUN/TURN 릴레이 | coturn (C, 관대한 라이선스). TURN-over-TCP 지원 추가 |
| 재사용 | WebRTC 전송(SRTP·ICE·DTLS) | libdatachannel (정적, `/MT`, 단일 exe) |
| 재사용 | 뷰어(디코딩·렌더) | 브라우저 네이티브 WebRTC + `<video>` |
| 자체 | 화면 캡처 | DXGI Desktop Duplication |
| 자체 | 색변환·다운스케일·인코딩 | D3D11/CPU 색변환 + Lanczos + Media Foundation H.264(HW+SW폴백) |
| 자체 | 세션/권한/보안 데스크톱 | 우리 소유 (롱테일 핵심) |
| 자체 | 입력 주입·커서 | SendInput + 별도 커서 채널 |
| 자체 | glue + manager-web 뷰어 UI | 우리 소유 |

초기 계획의 GStreamer(webrtcbin)는 채택하지 않았다. libdatachannel이 미디어 트랙(SRTP)을 기성 제공해 전송을 안 만들고 파이프라인 전체를 바로 검증할 수 있었고, 캡처·인코딩은 시스템 API(DXGI/MF)로 직접 구현하는 편이 무의존·단일 exe 원칙에 맞았다.

## 코덱 선택 — H.264 end-to-end

- 고객 fleet 전제: 외장 그래픽 없는 5년 내(2021+) 인텔 사무용 PC → 내장 Quick Sync가 H.264 하드웨어 인코딩을 전 노드에서 커버. 피제어 CPU 낮음.
- 웹 뷰어 디코딩까지 감안하면 브라우저·하드웨어 호환이 가장 넓은 H.264가 안전(H.265는 브라우저 지원 들쭉날쭉).
- AV1 제외 — 하드웨어 AV1 인코딩은 최신 극소수(Core Ultra 급)에만 있어 fleet 전체 미지원.
- 인코딩은 Media Foundation 하드웨어 MFT(D3D11 텍스처 제로카피) 우선, 하드웨어 불가 환경(RDP/VM)은 소프트웨어 MFT 자동 폴백 → 모든 환경 동작 보장.

## 서버 / 포트 설계

- coturn 위치 = Orange 서버(고객 폐쇄망 내). 대상 노드가 이미 MQTT로 outbound 하는 그 지점이라 자연히 닿음.
- 제어 포트 = **3184** (UDP+TCP 고정). 기존 318x 시리즈와 정렬: 3181(REST-API)·3182(MQTT TCP)·3183(MQTT WS)·3184(STUN/TURN).
- 릴레이 미디어 = UDP **3200–3300** (coturn min-port/max-port로 좁힘). TURN은 세션마다 릴레이 포트를 할당하므로 3184 하나로 못 합침.

고객사 방화벽 요청(대상·관제 세그먼트 → coturn, 전부 내부):

| 포트 | 프로토콜 | 용도 |
|------|----------|------|
| 3184 | UDP + TCP | STUN/TURN 제어 |
| 3200–3300 | UDP | 영상 릴레이 |

**both-relay(양방향 TCP 릴레이)**: 완전 UDP 차단 고객사는 outbound TCP 3184 하나만 열어도 동작한다. 노드와 관제(브라우저)가 각각 같은 coturn(:3184)에 TCP로 릴레이를 할당하고, coturn이 두 릴레이 할당을 자기 내부에서 이어준다(bridge). 릴레이 포트 3200–3300은 두 할당이 같은 coturn 위에 있어 coturn 내부에서만 오가므로 노드·관제 방화벽을 넘지 않는다. UDP를 열 수 있으면 3200–3300 UDP로 더 빠른 릴레이(또는 P2P 직결).

## TURN-over-TCP 구현

libjuice에 TCP TURN을 직접 추가(libnice/glib 없이 단일 정적 exe 유지). 기업이 outbound UDP를 전면 차단해도 노드가 coturn:3184에 TCP로 붙어 릴레이하고, 미디어를 그 TCP 연결에 TURN ChannelData로 실어 보낸다. ICE 자동선택이라 UDP 열린 곳은 P2P/UDP로 빠르게, 막힌 곳만 TCP 릴레이로 폴백. 전송은 이후 Winsock2 IOCP(overlapped `WSASend`/`WSARecv`, 전용 워커, `CRITICAL_SECTION`)로 재구현(`turn_iocp.c`) — STUN/ChannelData self-framing 재조립을 워커 스레드가 담당하고, 완성된 메시지만 poll 스레드로 넘긴다.

## both-relay 종단 완성 — UDP 차단·다중 NAT 기업망

TCP-over-TURN 기초 위에서, 관제·노드 양쪽이 같은 coturn(:3184)에 TCP 릴레이를 할당하고 coturn이 둘을 브리지하는 both-relay 경로를 실 노드(사무실 PC + 다중 NAT VM)에서 종단 동작시켰다. 사무실 PC만 되고 더 깊은 NAT VM은 안 되던 증상을 파고들어 **서로 독립적인 3개 근본원인**을 각각 잡았다.

1. **coturn 이중 인증 → 서명 없는 SUCCESS** — coturn에 `use-auth-secret`와 `lt-cred-mech`가 함께 켜져, 정적 크레덴셜 Allocate에 MESSAGE-INTEGRITY 없는 SUCCESS를 돌려줬다. libjuice가 "Missing integrity"로 할당을 거부 → 릴레이 후보 자체가 안 생김. coturn을 `use-auth-secret` 단일로 정리. 노드·브라우저 모두 임시 크레덴셜 사용.
2. **rest-api가 STUN 항목을 `username:null`로 직렬화 → 빈 크레덴셜 전달** — 크레덴셜 응답의 STUN IceServer가 username/credential을 null로 내보냈다. 노드측 agent가 "두 키가 있는 첫 항목"을 TURN 크레덴셜로 쓰는데, null이어도 키가 존재하면 빈 문자열로 뽑혀 STUN 항목을 잘못 골라 빈 username으로 Allocate → coturn 400 Bad Request. rest-api에 `response_model_exclude_none=True` → STUN 항목엔 두 키가 빠져 TURN 항목이 선택된다.
3. **Chrome이 ICE tiebreaker 0 전송 → libjuice가 검사를 400으로 거부** — Chrome이 릴레이로 보내는 연결성 검사의 `ICE-CONTROLLED` 속성 값(tiebreaker)이 0이었다. libjuice는 tiebreaker 0을 "속성 없음"으로 취급해 검사를 400으로 버렸고 → 유효 페어가 안 생겨 ICE가 완성되지 않았다. libjuice STUN 파서에서 파싱된 값이 0이면 1로 올려 존재 표시(`stun.c`). 실제 와이어 바이트로 확정.

지금은 relay-only(TCP)로 운영해 검증했다. 같은 망 직접 P2P(host) 최적화는 이 안정 기준선 위에 별도로 얹는다.

## 요구 재정의 — 동영상 부드러움이 1차 가치

- 최초 요구처 = 한 지자체 고객(전국 지자체 동일 니즈 → 시장 = 전국 지자체 = WAN 대규모 분산).
- 용도: 현장 PC에 달린 웹캠 실시간 영상을 원격으로 보는 것. 캠 전용 솔루션이 없어, 캠 영상이 떠 있는 PC 화면을 원격으로 본다.
- 따라서 원격 대상이 실시간 카메라 영상(풀모션) → **동영상 부드러움 = 1차 제품 가치**. "정지·문서 화면 위주" 가정은 폐기.

## 구현 현황

### 파이프라인 (전부 자체 코드 / 시스템 API, 외부 미디어 라이브러리 없음)

- **캡처**: DXGI Desktop Duplication (메인 모니터 1개 → 거울 효과 회피). 다중 모니터 전환 지원.
- **색변환**: GPU(D3D11 VideoProcessor, BGRA→NV12) / GPU 없는 환경은 CPU 색변환 폴백(풀범위 BT.709).
- **다운스케일**: 커스텀 Lanczos-2 셰이더(GPU) / CPU nearest(폴백).
- **인코딩**: Media Foundation H.264 — 하드웨어 MFT 제로카피 시도, 불가 시 소프트웨어 MFT 자동 폴백.
- **전송**: libdatachannel H.264 미디어 트랙(SRTP) → 브라우저 하드웨어 디코딩.

### 재생 부드러움 (풀모션 핵심)

- **FIFO 순서 전달** — P-프레임 유실 방지 → 움직임 깨짐 해결. 버퍼 초과 시 키프레임 강제 복구.
- **실제 캡처 시각 타임스탬프 페이싱** — 고정 간격 폐기 → 변동 fps여도 일정 속도 재생.
- **`timeBeginPeriod(1)` 타이머 정밀도** — Windows 기본 ~15.6ms 타이머로 sleep이 과다 수면하던 것 해소(22→30fps).
- **오디오 인터리브 제거**(비디오 전용) — placeholder 오디오가 비디오를 주기적으로 밀던 요인 제거.

### 화질·적응

- **전송 품질 QP 제어** — 높음/자동/낮음 모드가 해상도가 아니라 인코더 양자화(QP 상한)를 조정해 실제 전송 화질을 좌우.
- **적응형 비트레이트(AIMD)** — 브라우저 getStats 수신 손실 → MF 비트레이트 런타임 조정(0.5~40Mbps).
- **자동 해상도 매칭** — 뷰어 표시 크기에 맞춰 인코딩 해상도 조정 → 브라우저 재스케일 제거 + 대역폭 절감.
- **CPU 게이팅 적응** — `GetSystemTimes`로 진짜 포화(>85%)일 때만 다운. 가독성 우선: fps 먼저 낮추고 해상도 유지(하한 720). VM처럼 CPU는 노는데 GPU가 느린 경우 풀 해상도 유지 + fps만↓.
- **회전 모니터 대응** — 세로(회전 90°) 모니터는 획득 텍스처가 네이티브 가로라 크기 불일치로 검은 화면이 나던 것을, 네이티브 크기 캡처 + VideoProcessor 회전으로 해결.

### 상주 에이전트 부하 안정

- **백프레셔** — 큐 임계 초과 시 캡처·인코딩 스킵. 뷰어 이탈/정지/정체 시 즉시 idle.
- **send-on-change** — 화면이 안 바뀌면 재인코딩·전송 스킵(정지화면 enc 56→1~2fps, 대역폭 ~43%↓).
- working set 300MB대 안정(누수 아님 — WebRTC send 버퍼 + D3D11 텍스처 + 인코더 버퍼).

### 입력·커서

- **입력 주입** — 브라우저 정규화 좌표(0~1) → SendInput 절대좌표(0~65535). object-fit 레터박스 기준 매핑. 이동 스로틀(60Hz 코얼레스, 클릭은 항상 즉시).
- **커서** — 영상에 합성하지 않고 `GetCursorInfo` 125Hz 별도 채널 전송. 제어 중=로컬 커서(즉시 반응), 관망=원격 커서 오버레이.

## 로그인 안 된 화면 — 2프로세스 분리구조

로그인/잠금/UAC/보안 데스크톱까지 재연결 갭 없이 제어하기 위해, 단일 exe가 인자에 따라 두 역할을 한다.

- **remote 프로세스** (`--service`, Session 0 상주): WebRTC 연결·시그널링·데이터채널을 계속 쥔다. 캡처가 파이프로 보낸 H.264를 트랙으로 릴레이. 세션 전환 시 캡처만 교체(연결 유지 → 무이음).
- **캡처** (`--capture`, 콘솔 세션): `DxgiMfSource`로 캡처+인코딩 → 파이프. 입력 주입·커서 담당.

**IPC** — overlapped 단일 duplex 파이프. 처음엔 동기 단일 파이프였다가 read가 write를 막는 교착이 나서 파이프 2개로 나눴는데, 이건 반창고였다. 근본 해결은 `FILE_FLAG_OVERLAPPED`로 여는 것 — 그 락이 안 생겨 한 핸들에서 프레임 쓰기와 제어 읽기가 락 없이 동시 진행된다. 모든 대기는 op 이벤트 + stop 이벤트를 함께 기다려 종료 시 즉시 풀림.

### 세션·데스크톱 전환 견고성 (완료 · 리서치 검증)

- **아키텍처 검증**: 우리 분리구조 = Chrome Remote Desktop과 동일한 정답 토폴로지. 네트워크(WebRTC)를 쥔 영속 프로세스 + 세션마다 갈아끼우는 캡처 헬퍼. 세션이 바뀌어도 소켓·코덱·키가 안 끊긴다 → 재접속이 아니라 짧은 프레임 갭. 반대로 RustDesk는 연결을 헬퍼에 둬서 사용자 전환 때 재접속이 보인다(우리가 피한 함정).
- **핵심 원칙**: 세션 ID 변경만 헬퍼 재기동을 유발한다. 데스크톱 전환(lock/unlock/UAC)은 전부 같은 헬퍼 안에서 복제 재생성 + 주입 스레드 데스크톱 재부착으로 처리.
- **이벤트 기반 전환 감지**(폴링 폐기): 세션 변화는 `WTSRegisterSessionNotification` + 메시지 펌프 → `WM_WTSSESSION_CHANGE`. 데스크톱 전환은 `SetWinEventHook(EVENT_SYSTEM_DESKTOPSWITCH)` → 전환 즉시 재부착 + 복제 재생성.
- **DXGI 전환기 에러 처리**: `0x887A0025`(모드 변경 진행) 짧게 재시도, `0x887A0026`/`0x887A0001`(복제 무효화) 해제 후 재생성, `0x887A0022`(동시 duplication 4-슬롯 한도).
- **반영한 결함 수정**: 전환 순간 새 캡처 헬퍼의 최초 `initCapture`가 `DuplicateOutput` 일시 실패 한 방에 죽어 재기동 storm이 나던 것을, device는 이미 생성됐으니 값싼 복제 호출만 재시도하도록 수정. 옛 헬퍼 종료 대기로 복제 겹침 방지. send-on-change가 정지 화면의 첫 프레임을 삼키던 버그 수정.

### 잠금 화면 입력 주입

`SendInput`은 호출 스레드가 붙은 데스크톱에 주입된다. 주입 스레드도 SendInput 직전 현재 입력 데스크톱(Winlogon)에 부착 + 보안 데스크톱용 스캔코드(`KEYEVENTF_SCANCODE`) → 잠금 화면에서 마우스·암호 입력 → 원격 로그인까지 가능. 확인 완료.

## 배포·운영

- **DLL 0개 단일 exe** — 정적 OpenSSL + `/MT` + libdatachannel 정적(약 7.2MB, 시스템 DLL만 의존). vcpkg mbedTLS는 DTLS-SRTP 결여로 배제.
- 자체 STUN/TURN 등록 + 에이전트 자기 방화벽 등록(`netsh`, 관리자 권한 시) → 수동 설정 없이 직접 P2P 시도.
- 홀수 해상도 짝수 정렬(`MF_E_INVALIDMEDIATYPE` 회피). 로그 UTF-8·영문·기본 조용.

## 전략적 이점

- AGPL·Pro 라이선스 baggage 제거. libdatachannel(MPL-2.0)은 파일 단위 카피레프트라 정적 링크·비수정 사용은 무해.
- 우리 서명·우리 제품이라 백신 대응이 남의 RMM보다 깔끔.
- 로그인 안 된 보안 데스크톱 제어는 상용도 대개 유료/미지원 — 우리는 확보.

## 남은 항목

- 하드웨어 인코딩 경로 검증 — 실제 인텔 iGPU 엔드포인트에서 화질·CPU·fps 실측(RDP/VM은 하드웨어 인코더가 막혀 소프트웨어 최악 케이스로만 검증됨).
- 같은 망 직접 P2P(host) 최적화 — both-relay 안정 기준선 위에 host 후보 직접 연결을 얹어 릴레이 왕복 지연 제거.
- 커서 모양 전송(`GetFramePointerShape`), Ctrl+Alt+Del(`SoftwareSASGeneration`).

---

*Orange Platform 원격 화면 제어 솔루션 자체 구현 기술 리포트입니다.*
