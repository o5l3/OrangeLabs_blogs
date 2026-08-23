---
title: "덤프 분석에서 ntdll.dll이 원인으로 나온다면 — 예외 시점 컨텍스트로 콜스택 걷기"
excerpt: "프로세스 크래시 덤프를 분석하면 원인이 자꾸 ntdll.dll로 나온다. ntdll은 검출 지점이지 원인이 아니다. 덤프 기록 시점이 아니라 .ecxr(예외 시점 컨텍스트)에서 콜스택을 걸어야 진짜 원인을 짚을 수 있다는, 7개 증적 덤프로 확인한 트러블슈팅."
category: tech
date: 2026-08-19
author: kim-tigerj
tags: [크래시덤프, ntdll, WinDbg, ecxr, 예외처리, DbgEng, Windows, Orange Platform]
---

## 개요

비정상 종료 덤프를 분석하면 원인이 ntdll.dll로 나오는 건이 많다. 최근 30일 증상 12건 중 4건이 그랬다. 하지만 ntdll은 **검출 지점이지 원인이 아니다** — ntdll을 호출한 쪽이 진짜 문제다.

## 원인

콜스택을 **덤프 기록 시점의 스레드 컨텍스트**로 걷고 있었다. 크래시한 스레드는 덤프가 기록되는 그 시점에 WER(Windows Error Reporting) 대기 중이라, 스택이 전부 ntdll / kernelbase / kernel32로 채워져 있다. 모든 프레임이 OS 코어면 가장 안쪽의 ntdll이 원인으로 확정되고, 정답을 아는 `ExceptionAddress` 수집기는 "cause가 이미 있으니" 스킵된다.

증적 덤프 7건(WmiPrvSE, vmnat, kcppayservice, AdobeCollabSync, picpick, comvictim, rpcvictim)을 `.ecxr`(예외 시점 컨텍스트)로 걸어보면 7건 전부 진짜 원인이 드러난다. 32비트 덤프까지 포함해서다.

과거에는 선정 로직(5단 캐스케이드, RaiseException 패치, ModuleTier)을 계속 손봤지만, **스택을 어느 컨텍스트에서 걷는지**는 한 번도 고치지 않았다. 당시의 코퍼스는 스택 깊은 곳에 서드파티 범인 프레임이 우연히 남아 있어서 결함이 가려져 있었던 것이다.

## 수정

- 유저모드도 예외 시점 컨텍스트로 스택을 걷는다 (`GetStoredEventInformation` → `GetContextStackTrace`). 실패하면 기존 `GetStackTrace`로 폴백한다. 커널 0x3B / 0x7E와 같은 패턴이다.
- 콜스택 경쟁에서 OS 코어만 남으면 원인으로 확정하지 않는다. `ExceptionAddress` 수집기가 잡게 넘긴다.
- NoFilter 폴백이 유저모드에서 OS 코어 모듈을 반환하지 않게 한다. 최후 폴백은 비정상 종료한 EXE 자신이다.
- 커널(BSOD) 경로는 건드리지 않는다.

## 결과 (증적 7건)

| 덤프 | 수정 전 | 수정 후 |
|---|---|---|
| WmiPrvSE | ntdll.dll | WmiPrvSE.exe |
| vmnat | ntdll.dll | vmnat.exe |
| comvictim | ntdll.dll | comvictim.exe |
| rpcvictim | ntdll.dll | rpcvictim.exe |
| picpick | ntdll.dll | f_sps (서드파티 DLL) |
| kcppayservice | ntdll.dll | kcppayservice.exe |
| AdobeCollabSync | ntdll.dll | AdobeCollabSync.exe |

picpick 케이스가 특히 의미 있다. 원인이 EXE 자신이 아니라 그 안에 로드된 서드파티 DLL(f_sps)로 정확히 좁혀졌다. "ntdll이 원인"이라는 무의미한 결론이, 예외 시점 컨텍스트 하나만 제대로 걸어도 "이 서드파티 모듈이 원인"이라는 실행 가능한 진단으로 바뀐다.

## 교훈

크래시 덤프에서 스택이 전부 OS 코어(ntdll/kernelbase)로 나온다면, 그것은 원인이 아니라 **덤프를 기록한 시점의 정지 상태**일 가능성이 높다. 진짜 원인은 예외가 발생한 순간의 컨텍스트(`.ecxr`)에 있다. 선정 휴리스틱을 아무리 정교하게 다듬어도, 애초에 잘못된 컨텍스트에서 스택을 걸으면 답이 나오지 않는다.

*Orange Platform 에이전트 덤프 분석기의 프로세스 크래시 원인 특정 트러블슈팅 리포트입니다.*
