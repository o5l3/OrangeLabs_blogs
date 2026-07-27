---
title: "Driver Verifier가 드러낸 커널 BSOD — 락을 쥔 채 ZwClose를 부르면 안 되는 이유 (0xC4 IrqlZwPassive)"
excerpt: "Driver Verifier(/standard /driver) 활성 상태에서 드라이버 언로드 시 0xC4 DRIVER_VERIFIER_DETECTED_VIOLATION BSOD가 발생한 사례. ZwClose가 PASSIVE_LEVEL 전용 API인데 fast mutex를 쥔 APC_LEVEL에서 호출되던 구조적 결함을 WinDbg로 규명하고, 락 밖에서 핸들을 닫도록 정리 루프를 재설계한 기록."
category: tech
date: 2026-07-22
author: wychoi-orangelabs
tags: [커널 드라이버, Driver Verifier, BSOD, WinDbg, ZwClose, IRQL, fast mutex, Windows, Orange Platform]
---

## 개요

Driver Verifier(`verifier /standard /driver orange.sys`)를 활성화한 상태에서 `sc stop orange`로 드라이버를 언로드하면 BSOD가 발생했다. Verifier 없이 평소에는 통과하던 코드였지만, 실제로는 커널 API 호출 규약을 위반하는 구조적 결함이 숨어 있었고 Verifier가 그것을 표면으로 끌어낸 것이다.

## 문제

### 버그체크

- 코드: `0xC4 DRIVER_VERIFIER_DETECTED_VIOLATION`
- Arg1: `0x0012001F` = DDI 규칙 `IrqlZwPassive` (DDI = Device Driver Interface, 드라이버가 지켜야 하는 커널 API 호출 규약)
- 위반 조건: `ZwClose should only be called at IRQL = PASSIVE_LEVEL`
- 크래시 시점 IRQL: `APC_LEVEL(1)` (`PASSIVE_LEVEL=0`이 정상이며, `ZwClose`는 0에서만 호출 가능)
- 대상 바이너리: orange.sys File version 0.3.1.35 (Debug 빌드)

### 콜스택

```
orange!CProcessTable::CloseProcessHandle+0x1c   CProcessTable.h:230   (ZwClose 호출)
orange!CProcessTable::FreeEntryData+0xed        CProcessTable.h:1447
orange!CProcessTable::Clear+0x90                CProcessTable.h:1474
orange!CProcessTable::Destroy+0x4f              CProcessTable.h:137
orange!DestroyProcessTableThread+0x15           Process.cpp:26
nt!PspSystemThreadStartup / nt!KiStartSystemThread
```

### 근본 원인

`CProcessTable::Clear()`가 테이블 락(`Lock()`)을 쥔 채 루프를 돌며 각 엔트리에 대해 `FreeEntryData()` → `CloseProcessHandle()` → `ZwClose(hProcess)`를 호출한다.

- 전역 인스턴스 `g_process`는 `m_bSupportDIRQL == 0`이라 `Lock()`이 `ExAcquireFastMutex()` 경로를 탄다. fast mutex는 IRQL을 `APC_LEVEL(1)`로 올린다.
- `ZwClose`는 `PASSIVE_LEVEL` 전용 API이므로 `APC_LEVEL`에서 호출하면 규약 위반이다.
- `hProcess`는 프로세스 등록(`Add3`) 시 `ZwOpenProcess(OBJ_KERNEL_HANDLE)`로 연 커널 핸들이다. 언로드 시 `Clear`가 테이블의 모든 엔트리를 정리하며 그 핸들들을 락 안에서 `ZwClose` 하므로 재현율은 100%다.
- 참고로 `m_bSupportDIRQL == 1`(스핀락, `DISPATCH_LEVEL=2`)이어도 `PASSIVE`를 초과하므로 동일 위반이다.

즉, **락의 종류가 문제가 아니라 "락을 쥔 채 `ZwClose`를 부른다"는 구조 자체가 원인이다.** Verifier 없이도 원칙상 불법이며, 평소에 우연히 통과하던 것을 Verifier가 드러낸 것뿐이다.

### 증거 (WinDbg)

- `!analyze -v`: `DV_VIOLATED_CONDITION = ZwClose should only be called at IRQL = PASSIVE_LEVEL`
- `!irql` 및 `r cr8` = 1 (APC_LEVEL) — x64는 현재 IRQL을 CR8 레지스터에 저장하며 크래시 컨텍스트에 그대로 보존된다.
- `dt orange!CProcessTable g_process`: `m_bSupportDIRQL = 0`(원시 바이트 00 확인) → fast mutex 경로 확정.
- `dt nt!_FAST_MUTEX`: `Owner` = 크래시 스레드, `OldIrql = 0` → 우리 스레드가 fast mutex를 쥔 채(IRQL 0→1) 진입했음을 확인.

## 해결

### 수정안

`Clear()`를 같은 클래스의 검증된 `Remove()` 패턴으로 변경한다. **락 안에서는 엔트리 복사 + 노드 삭제만 하고, `ZwClose`와 풀 해제는 락을 푼 `PASSIVE` 상태에서 수행**한다.

```cpp
void Clear()
{
    if (false == IsPossible() || false == m_bInitialize) return;

    // 시작 원소 수 + 여유를 상한으로 (동시 삽입 폭주/무한루프 방어)
    ULONG guard = 0;
    { KIRQL irql = 0; Lock(&irql); guard = RtlNumberGenericTableElements(&m_table); Unlock(irql); }
    guard += 16;

    bool bGot = true;
    for (ULONG i = 0; bGot && i < guard; i++) {
        PROCESS_ENTRY entry = { 0, };
        KIRQL         irql  = 0;
        bGot = false;

        Lock(&irql);
        if (!RtlIsGenericTableEmpty(&m_table)) {
            PPROCESS_ENTRY p = (PPROCESS_ENTRY)RtlGetElementGenericTable(&m_table, 0);
            if (p) {
                entry = *p;                                      // 소유권(핸들/버퍼)을 로컬로 이전
                if (RtlDeleteElementGenericTable(&m_table, p))   // 노드만 해제
                    bGot = true;
            }
        }
        Unlock(irql);

        if (bGot) FreeEntryData(&entry);   // ZwClose + 풀 해제를 락 밖(PASSIVE)에서
    }
    if (bGot) __log("%s: guard limit reached", __FUNCTION__);   // 상한에서 끊김 = 이상 신호
}
```

### 포인트

- `entry = *p` 전체 복사로 소유권(`hProcess`/`ProcPath`/`Command` 등)을 로컬로 이전한다. `RtlDeleteElementGenericTable`의 Free 콜백은 `CMemory::Free(node)`로 노드만 해제하므로 더블 프리·누수가 없다.
- 삭제 실패 시(`bGot=false`)에는 복사본을 해제하지 않고 종료해 더블 프리를 방지한다.

### 검증 방법

1. Debug 빌드 → VM에 orange.sys 재설치
2. `verifier /standard /driver orange.sys` → 재부팅
3. `sc stop orange` → BSOD 미발생이면 해결
4. `verifier /query`로 계측 활성 확인 후 부하 테스트

### 추가로 확인할 지점

- 언로드 진행 중 프로세스 테이블 삽입(`Add3` 등)이나 프로세스 알림 콜백이 차단되는지 — 차단돼 있어야 락을 푼 틈의 동시 삽입 없이 루프가 안전하게 종료된다.
- `Clear` 호출부가 언로드(PASSIVE) 경로뿐인지 — `DISPATCH`/`APC`에서 불릴 경로가 있으면 `ZwClose`를 락 밖으로 빼도 여전히 위반이다.
- `Remove_OLD()`도 락 안에서 `FreeEntryData`를 호출하는 동일 버그가 있어, deprecated 여부에 따라 방치/삭제를 판단해야 한다.

---

*이 글은 Orange Platform 에이전트 커널 드라이버의 BSOD 분석 및 트러블슈팅 리포트입니다.*
