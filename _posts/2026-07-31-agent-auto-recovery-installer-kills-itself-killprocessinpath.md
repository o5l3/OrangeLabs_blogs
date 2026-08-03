---
title: "자동 복구가 한 번도 동작하지 않았다 — 복구 인스톨러가 자기 부모를 종료시킨 결함"
excerpt: "Windows 에이전트의 자동 복구(--recover)가 처음부터 동작하지 않던 원인을 추적한 기록. 복구 인스톨러가 기존 설치 정리(--remove) 단계에서 데이터 경로 하위 프로세스를 통째로 kill하는데, 자기 자신을 실행시킨 부모 인스톨러가 마침 그 경로(Packages\\)에 있어 스스로를 죽이고, 파일 복사도 --install도 못 한 채 10분마다 무한 반복하던 결함을 규명하고 수정 방안을 도출했다."
category: tech
date: 2026-07-31
author: kim-tigerj
tags: [Windows, 자동 복구, 워치도그, 인스톨러, 프로세스 종료, C++, 트러블슈팅, Orange Platform]
---

## 요약

자동 복구(`orange.exe --recover`)는 **처음부터 동작하지 않았다.** 복구를 수행하는 인스톨러가 복구 첫 단계에서 자기 자신(정확히는 자신을 실행시킨 부모 인스톨러)을 종료시키기 때문이다.

워치도그가 실제로 발동하게 되면서 이 결함이 드러났다. 그전에는 워치도그가 제대로 돌지 않아 복구 경로 자체가 실행될 일이 드물었고, 그래서 오래 감춰져 있었다.

## 원인 (확정)

연쇄는 다음과 같다.

1. 워치도그가 서비스 부재를 감지 → `orange.exe --recover` 실행.
2. `Recover()`가 Good 패키지 인스톨러를 `/S`로 실행 (`main.cpp:1039`). 그 인스톨러 경로는 `C:\ProgramData\Orange\Packages\<version>\ORANGE.x64.exe`.
3. 인스톨러가 기존 설치 정리를 위해 `--remove`를 실행.
4. `--remove` 안의 `ClearData()`가 두 경로의 프로세스를 종료한다 (`main.cpp:1069-1070`):

   ```cpp
   YAgent::KillProcessInPath(p->AgentPath()->path.c_str());   // C:\Program Files\ORANGE
   YAgent::KillProcessInPath(p->AgentPath()->data.c_str());   // C:\ProgramData\Orange
   ```

5. `KillProcessInPath()`는 **하위 디렉터리까지 포함해 매칭**한다 (`common/Process.cpp:483-486`). `Packages\<version>\`은 `C:\ProgramData\Orange`의 하위다.
6. 자기 보호는 `dwCurrentProcessId` 하나뿐이다 (`common/Process.cpp:462`). 현재 프로세스는 `%TEMP%`에서 도는 `orange.exe`이므로 살아남지만, **자신을 실행시킨 부모 인스톨러는 보호 대상이 아니라 그대로 종료된다.**
7. 인스톨러가 죽었으므로 파일 복사도 `--install`도 수행되지 못한다 → 서비스 복원 실패.
8. 10분 뒤 워치도그가 재시도 → 같은 지점에서 또 사망 → 무한 반복.

## 관측 증거 (2026-07-30)

`agent.install.log` — 10분 간격으로 `--remove` 블록만 반복되고 `--install` 항목이 전혀 없다.

```text
17:50:04 --remove   C:\ProgramData\Orange\Packages\1.6.242.259\ORANGE.x64.exe
17:50:04   ... service Remove skipped (not STOPPED) / CFilterCtrl uninstalled
17:50:04   ClearData begin
17:50:04   ClearData end
18:00:04 --remove   C:\ProgramData\Orange\Packages\1.6.242.259\ORANGE.x64.exe
        (이하 동일 반복)
```

`agent.recover.log` — 매 주기 Good 인스톨러 실행까지는 정상 도달한다.

```text
18:00:02 Recover  good 1.6.242.259
18:00:02   Installer C:\ProgramData\Orange\Packages\1.6.242.259\ORANGE.x64.exe
18:00:03   path ... / arg /S
```

Windows 이벤트로그(Orange, EventId 1717) — 워치도그가 10분마다 정상 발동 중.

`--remove` 프로세스는 `%TEMP%`에서 실행되어 데이터 경로 밖이므로 `ClearData end`까지 정상 기록을 남긴다. 죽는 것은 부모 인스톨러다. 로그가 "제거는 되고 설치는 안 되는" 모양인 이유가 이것이다.

## 영향 범위

- **자동 복구 전체** — Packages 폴더의 인스톨러를 실행하는 모든 경로가 동일하게 실패한다. Good 버전이 무엇이든 무관하다.
- **워치도그 3단 복구** — Good·Last 매체는 둘 다 `Packages\` 하위이므로 같은 방식으로 죽는다. 3차 서버 재다운로드분만 `%TEMP%`에 받으므로 유일하게 살아남는다.
- **실사용 영향**: 에이전트가 손상되거나 서비스가 사라진 노드는 자동 복구되지 않고 10분마다 `--remove`만 반복한다. 사람이 수동 재설치해야 한다.

## 수정 방안

| 안 | 내용 | 평가 |
|---|---|---|
| **1. 인스톨러를 %TEMP%로 복사 후 실행 (권고)** | `Recover()`가 Packages의 인스톨러를 직접 실행하지 않고 `%TEMP%`에 복사해 실행. 데이터 경로 밖이라 청소 사정권에서 벗어난다 | 변경 범위 최소. 인스톨러의 `--remove` 호출이 이미 같은 이유로 TEMP 복사 방식을 쓰고 있어 관례도 일치 |
| 2. ClearData에서 데이터 경로 kill 제외 | `KillProcessInPath(data)` 호출 제거 | 도입 이력과 목적(고아 프로세스 정리 등)을 확인해야 하며 부작용 우려 |
| 3. KillProcessInPath에 부모·조상 프로세스 예외 추가 | 범용 수정 | 이 함수를 쓰는 다른 호출부에 영향 |

**1안을 권고한다.** 워치도그의 복구 매체 선택부에도 동일하게 적용해야 Good·Last 경로가 살아난다.

## 왜 지금까지 발견되지 않았나

워치도그가 실제로 발동한 적이 거의 없었다. 워치도그 작업은 `--install` 시점에만 등록되어 스크립트가 갱신되지 않았고, 서비스가 사라진 상태 자체가 드물었다. 워치도그를 정상화하자 10분마다 같은 지점에서 실패하는 양상이 로그에 또렷이 남았고, **"--remove는 있는데 --install이 없다"**는 차이가 실마리가 됐다.

*Orange Platform Windows 에이전트 자동 복구 결함 트러블슈팅 리포트입니다.*
