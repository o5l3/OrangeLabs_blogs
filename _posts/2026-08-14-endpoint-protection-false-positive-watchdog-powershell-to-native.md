---
title: "백신이 우리 워치도그를 드로퍼로 오탐하다 — PowerShell EncodedCommand를 네이티브 실행으로 걷어낸 이야기"
excerpt: "Symantec Endpoint Protection이 에이전트 워치도그 예약 작업을 Scr.Malcode!gen77로 격리했다. EncodedCommand + ExecutionPolicy Bypass + TLS 검증 무력화 + 무인 다운로드 실행이 정확히 드로퍼 휴리스틱에 걸린 사건과, PowerShell 게이트를 네이티브 코드로 옮겨 스크립트 자체를 없앤 대응 기록."
category: tech
date: 2026-08-14
author: kim-tigerj
tags: [보안오탐, EDR, PowerShell, 예약작업, Watchdog, 안티바이러스, Orange Platform]
---

## 사건

한 고객사에서 Symantec Endpoint Protection(SEP)이 에이전트 워치도그 예약 작업을 `Scr.Malcode!gen77`로 탐지·격리했다. 격리 대상은 `OrangeTheClientWatchdog`(`C:\Windows\System32\Tasks\` 하위의 작업 정의)였다.

우리 제품이 악성 스크립트를 심는 것처럼 보이는 상황이므로 신속한 처리가 필요했다.

## 원인 — 우리가 정말 드로퍼처럼 행동하고 있었다

워치도그 예약 작업은 동작 조건 판정을 `PowerShell -EncodedCommand`로 수행하고 있었다. 여기에 복구 매체 3단(Good → Last → 서버 재다운로드)을 얹으면서, 스크립트 내용이 드로퍼 패턴에 정확히 부합하게 됐다.

| 스크립트 요소 | 휴리스틱 판정 |
|---|---|
| `-EncodedCommand`(base64) + `-ExecutionPolicy Bypass` + `-WindowStyle Hidden` | 난독화·정책 우회 |
| `[Net.ServicePointManager]::ServerCertificateValidationCallback={$true}` | TLS 검증 무력화 |
| `Invoke-WebRequest -OutFile`로 실행 파일 내려받기 | 드로퍼 |
| 받은 실행 파일을 `/S`로 조용히 실행 | 무인 설치 |

"인증서 검증을 끄고 인터넷에서 실행 파일을 받아 조용히 실행한다" — 백신이 잡는 것이 오히려 정상이다.

**증상 증폭:** "에이전트 기동마다 워치도그 재등록"을 넣어 뒀기 때문에, 백신이 작업을 격리하면 다음 기동 때 다시 등록되고 백신이 또 격리한다. 탐지 경보가 반복되는 악순환이 됐다.

사전 경고도 있었다. 교차검증 단계에서 "EncodedCommand 자체가 보안 제품에 고위험으로 차단될 수 있다", "주 실행 파일과 독립된 최소 네이티브 실행 파일이 더 강한 격리를 제공한다"는 지적이 나왔으나, 장기 과제로 미뤄 기록만 해 둔 상태였다.

## 처리 방침 — PowerShell 완전 제거

예약 작업의 동작을 PowerShell이 아닌 `orange.exe --recover` 직접 실행으로 바꾼다. 스크립트가 사라지므로 스크립트 휴리스틱 자체가 적용될 여지가 없어진다.

### 선행 필수 — 게이트를 orange.exe 안으로 옮긴다

문제는 `--recover`에 현재 상태 검사가 없다는 점이었다. 인스톨러 뮤텍스 확인 후 곧바로 `Recover()`를 호출하며, `Recover()`는 Good 패키지가 있으면 무조건 인스톨러를 `/S`로 실행한다. "언제 복구할지"의 판단은 전부 PowerShell 게이트 안에 들어 있었다.

따라서 게이트 없이 주기 실행으로 바꾸면 정상 장비에서도 주기마다 전체 재설치가 돈다. 재설치는 `--remove`로 시작하므로 에이전트가 계속 끊겼다 살아난다. **반드시 게이트를 먼저 네이티브로 옮겨야 했다.**

| 게이트 항목(기존 PowerShell) | 네이티브 대체 |
|---|---|
| `ShutdownCause == uninstall`이면 중단 | `GetShutdownCause()` — `--logon`에서 이미 사용 |
| 서비스 없음 / `Start != 2` | `service.IsInstalled()` + 레지스트리 `Start` |
| 서비스 10분 이상 정지 | `IsRunning()` + 최초 정지 시각을 레지스트리에 기록해 경과 판정 (7036 이벤트 조회보다 단순·견고) |
| 인스톨러 뮤텍스 | 이미 있음 |

정지 경과 판정이 필요한 이유: 업데이트·재시작 중 서비스가 잠깐 멈춘 순간에 복구가 발동하면 안 된다.

### 서명 기반 손상 판정은 이번에 넣지 않는다

"orange.exe가 손상됐는지" 판정은 외부 판정자가 필요하나, 이번 전환에서는 제외했다. 실효가 거의 없기 때문이다.

- 손상된 orange.exe는 실행 자체가 안 되므로 판정 이전에 무력하다.
- 그 자동 복구는 실제로 한 번도 동작한 적이 없었다(복구 인스톨러가 스스로를 지우는 문제로, 이후 별도 수정됨).
- 설치 시점 방어(flush 확인 + 자기 무결성 검증)가 손상된 설치를 애초에 완료시키지 않는다.

판정자가 필요해지면 별도의 서명된 네이티브 실행 파일로 다루는 것이 맞다(후속 과제).

## 변경 범위

| 파일 | 변경 |
|---|---|
| `yagent21/main.cpp` | `--recover`에 게이트 추가 (uninstall 중단 / 서비스 정상이면 종료 / 정지 경과 판정) |
| `yagent21/CAgentRecover.h` | `RegisterWatchdogTask`의 Action을 `powershell.exe` → `orange.exe --recover`로. `BuildGuardArguments` 제거 |

작업 정의가 바뀌므로 `SameWatchdogArguments`의 Path 비교(`powershell.exe` ≠ `orange.exe`)가 자동으로 불일치를 잡아 재등록한다. 별도 조치는 불필요하다.

## 교훈

우리 코드가 정당한 의도로 짜여 있어도, **행동 지문이 악성 도구와 같으면 EDR은 의도를 보지 않는다.** base64 인코딩·정책 우회·TLS 검증 무력화·무인 다운로드 실행은 그 자체로 고위험 신호다. 자기 방어 로직을 스크립트로 짜는 대신 서명된 네이티브 실행 파일로 옮기는 것이, 오탐을 피하는 가장 근본적인 방법이다.

*Orange Platform 에이전트 워치도그의 보안 제품 오탐 대응 리포트입니다.*
