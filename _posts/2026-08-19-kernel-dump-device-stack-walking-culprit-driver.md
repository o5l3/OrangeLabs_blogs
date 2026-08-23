---
title: "0x9F 블루스크린에서 진짜 원인 드라이버 찾기 — 디바이스 스택 워킹으로 미니포트 특정"
excerpt: "0x9F(DRIVER_POWER_STATE_FAILURE)는 콜스택 폴백이 항상 ntoskrnl.exe로 나와 진짜 원인 드라이버를 못 짚는다. DEVICE_OBJECT/DRIVER_OBJECT를 직접 워킹해 PDB 없이도 원인 미니포트를 특정하고, 크래시를 10개 범주로 분류하며, WHEA 레코드를 파싱해 하드웨어 결함까지 짚어낸 덤프 분석기 개선기."
category: tech
date: 2026-08-19
author: kim-tigerj
tags: [BSOD, 블루스크린, 커널덤프, 디바이스스택, WinDbg, DbgEng, WHEA, Windows, Orange Platform]
---

## 개요

덤프 분석기가 BSOD(특히 0x9F)와 사용자 모드 프로세스 덤프 양쪽에서 **원인 모듈을 정확히 식별**하고, 운영자가 다음 액션을 즉시 결정할 수 있도록 **분류·하드웨어 정보를 함께 제공**하도록 개선했다.

핵심은 분석기와 룰의 역할을 명확히 가른 것이다. 분석기는 데이터 추출·식별·분류만 하고, 사용자 안내 문구는 룰이 담당한다.

## 배경 — 무엇이 문제였나

| 문제 | 영향 |
|---|---|
| 0x9F Arg2 = PDO — 직접 코드 주소 없음 | 콜스택 FALLBACK으로 항상 ntoskrnl.exe. 진짜 미니포트 식별 불가 |
| 0xEA Arg3 = 드라이버명 UNICODE_STRING* | 디참조 처리 없음 |
| 0x124 (WHEA_UNCORRECTABLE_ERROR) | Cause가 PSHED.dll / ntoskrnl.exe — 운영자에게 도움 안 됨 |
| ntkrnlmp.exe vs ntoskrnl.exe 혼재 | 룰 매칭 일관성 깨짐 |
| Unloaded 모듈 `<Unloaded_xxx>` | path 손실 → PE Version 못 읽음 |
| 사용자 모드 프로세스 크래시도 software로 분류 | 의미 부정확 (드라이버 결함 ≠ 응용 프로그램 크래시) |
| 카테고리/분류 정보 부재 | 룰이 BugCheck 코드만 보고 분기 |

가장 뿌리 깊은 문제가 첫 줄이다. 0x9F(DRIVER_POWER_STATE_FAILURE)는 인자로 코드 주소가 아니라 PDO(Physical Device Object) 포인터를 준다. 그래서 콜스택을 걸으면 결국 커널 코어(ntoskrnl.exe)에서 바닥나고, 진짜 전원 전환에 실패한 미니포트 드라이버는 끝내 드러나지 않았다.

## A. 0x9F / 0xEA 디바이스 스택 워킹

PDB(심볼)에 의존하지 않고 `DEVICE_OBJECT` / `DRIVER_OBJECT`를 직접 워킹하는 신규 모듈을 만들었다. 폐쇄망 현장에서도 심볼 없이 동작해야 하기 때문이다.

- PDO부터 `AttachedDevice` 체인을 위로 따라가며 미니포트를 식별
- PDB가 가능하면 `GetSymbolTypeId` + `GetFieldOffset`, 없으면 x64 표준 오프셋으로 폴백 (sanity < 0x1000)
- 인프라 드라이버 화이트리스트(pci/acpi/wdf/ndis/fltmgr 등)를 필터링, 사이클 감지, 포인터 정합성 검사
- `UNICODE_STRING` 디코더 (Length 홀수·MaximumLength 초과 거부)
- 0xEA는 Arg3를 디참조해 모듈에 매핑 (`name.sys` / `name` 둘 다 비교)
- **0순위 cascade preempt** — 콜스택 FALLBACK 이전에 0x9F/0xEA의 cause를 무조건 덮어쓴다

## B. Cause 분류 — 10개 범주, 항상 출력

BugCheck 코드만 보던 룰이 이제는 분석기가 확정한 범주로 분기할 수 있도록, Cause에 category를 붙였다.

| Value | 트리거 |
|---|---|
| process | IsUserMode() |
| kernel | nt/hal + BugCheck-특정 매핑 없음 |
| software | 그 외 일반 드라이버 |
| firmware | 0xA5 ACPI_BIOS_ERROR, WHEA Firmware |
| cpu | 0x9C MCE, 0x101 워치독, WHEA Processor |
| disk | 0x77/0x7A, WHEA storage PCIe |
| memory | WHEA Memory 섹션 |
| network | WHEA PCIe + network class |
| hardware | 0x124 (섹션 파싱 실패 시) |
| unknown | 식별 실패 |

**우선순위 정정**: 원래는 `IsAbsoluteKernelCore` 검사가 BugCheck-특정 매핑보다 먼저 와서, 0x124 + Cause=ntoskrnl 조합이 hardware가 아니라 kernel로 잘못 분류되고 있었다. 새 순서는 `IsUserMode → BugCheck-특정 → IsAbsoluteKernelCore → 기본 software`다.

## C. 하드웨어 정보 보강 — 키는 늘, 값은 조건부

Cause.Hardware[] 키는 항상 출력하되, 값이 채워지는 것은 아래 셋 중 하나일 때다. 빈 배열이 나오는 것이 정상이다.

| 조건 | 경로 | 라이브 PC 의존 |
|---|---|---|
| 0x124 WHEA + PCIe VEN/DEV 매치 성공 | 매치 드라이버의 PnP 디바이스 enumerate (SetupAPI) | O |
| 0x124 WHEA + 비-PCIe / PCIe 매치 실패 | 각 WHEA 섹션의 hardwareEntry (memory/cpu/pcie) | 덤프 자체 |
| category=software + cause가 SCM 서비스로 등록 | FindServiceByFileName → FindHardwareDevices | O |

라이브 PC 의존(O) 표시는 **분석 PC = 원본 PC**일 때만 적중한다. 외부 덤프를 다른 PC에서 분석하면 cause 드라이버가 분석 PC에 등록돼 있지 않아 빈 배열이 된다. 즉 에이전트가 자기 PC 덤프를 분석할 땐 채워질 가능성이 크고, 사후 포렌식(다른 PC에서 분석)에서는 거의 빈 배열이다. 룰은 "있으면 부가 표시, 없으면 무시"로 다뤄야 한다.

## D. 이름 정규화 + 원본 보존

- `ntkrnlmp.exe` / `ntkrnlpa.exe` / `ntkrpamp.exe` → `ntoskrnl.exe`
- `halmacpi.dll` / `halacpi.dll` → `hal.dll`
- 원본은 `Cause.originalName`으로 보존

이름이 혼재하면 룰 매칭이 깨지므로, 표준 이름으로 정규화하되 원본을 잃지 않도록 별도 필드에 남긴다.

## E. 0x124 WHEA 정밀 처리

`WHEA_ERROR_RECORD`(Arg2)를 직접 파싱한다. CPER 시그니처를 검증하고 → 섹션 디스크립터를 순회 → GUID를 매칭한다. 지원 섹션은 Memory / Processor(Generic + XPF) / PCIe / Firmware다. PCIe 섹션 + 라이브 PC에서 드라이버 식별에 성공하면 Cause를 PSHED 대신 실제 드라이버로 교체하고 category를 자동으로 software로 전환한다. 메모리 접근이 불가능하면 generic `hardware`로 정직하게 폴백한다.

## F. Unloaded 모듈 path 복구

DbgEng가 `<Unloaded_xxx>`로 표시하는 케이스를 라이브 검색으로 복구한다. 6단계다.

1. EXE와 같은 디렉터리
2. 하위의 흔한 폴더
3. SCM 서비스 맵
4. `System32\drivers`
5. SearchPath API
6. DriverStore 전체 enumeration

성공하면 path/exist/Version/timestamp를 갱신하고, 못 찾으면 graceful fallback한다. (커널 모듈 이름은 EXE 옆의 동명 파일 오인을 막기 위해 1단계를 스킵한다.)

## Report.DeviceStack[] 출력 예

0x9F처럼 PDO를 워킹한 케이스에서 채워진다. 그 외에는 빈 배열이다.

```json
{
  "device":     "0xffff920fc0752060",
  "driver":     "0xffff920fbf55ee10",
  "driverName": "\\Driver\\e1dnexpress",
  "module":     "e1dn.sys",
  "infra":      false,
  "culprit":    true
}
```

## 검증 — 원인불명 26개 코퍼스

| 카테고리 | 건수 | 대표 |
|---|---|---|
| process | 11 | Explorer, iexplore, PowerShell, delphi32 (vcldesigner70.bpl 포함) 등 |
| kernel | 10 | HYPERVISOR_ERROR×6, SYSTEM_THREAD_EXCEPTION×2 등 |
| software | 4 | e1dn.sys(0x9F), xfilter64.sys(DPC_WATCHDOG), CLASSPNP.SYS, orange.sys |
| hardware | 1 | WHEA (미니덤프 정밀 분류 불가) |

- Cause.category 26/26 보장, JSON 출력 누락 0건
- 0x9F + e1dn.sys — Cause·DeviceStack·culprit 모두 정확히 식별
- WHEA + ntoskrnl → 우선순위 정정으로 hardware로 정확히 분류
- 한계: WHEA 미니덤프는 generic `hardware`로 폴백한다. 다른 PC에서 분석할 때 path 복구가 실패하는 것은 정상이다(라이브 PC를 전제한 휴리스틱이기 때문).

## 룰이 어떻게 쓰는가

분석기가 category·Hardware·DeviceStack·originalName 같은 식별자를 제공하면, 룰은 그 값으로 안내 문구를 분기한다.

```json
// process — 응용 프로그램 크래시
{ "match": { "{cpptr.category}": { "$eq": "process" } },
  "Detail": [ "응용 프로그램 비정상 종료. 최신 버전 설치 또는 제조사 문의." ] }

// software + Hardware가 있을 때 풍부한 안내
{ "match": { "{cpptr.category}": { "$eq": "software" } },
  "Detail": [ "원인: 이 PC의 {cpptr.Hardware[0].description} 드라이버",
              "조치: 해당 드라이버 업데이트" ] }
```

분석기는 "무엇이 원인인가"를 데이터로 확정하고, 룰은 "그래서 사용자에게 뭐라고 말할 것인가"만 담당한다. 이 분담 덕분에 새 BugCheck 유형이 들어와도 분석기 코드를 건드리지 않고 룰만 추가하면 된다.

*Orange Platform 에이전트 덤프 분석기의 원인 드라이버 특정 기법에 관한 기술 리포트입니다.*
