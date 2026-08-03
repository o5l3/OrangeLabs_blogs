---
title: "마스터 이미지 복제 환경의 PC명 중복과 다운로드 최적화(Delivery Optimization) 분석"
excerpt: "콜센터 고객사 환경에서 다수 PC의 Windows 컴퓨터 이름(hostname/NetBIOS)이 중복 설정된 현상과 Windows 다운로드 최적화(Delivery Optimization) 설정을 분석한 리포트. 두 사안 모두 마스터 이미지 복제로 동일 사양 단말을 다수 배포한 공통 원인에서 파생되며, DO 피어 식별은 PC명이 아닌 랜덤 GUID 기반임을 규명하고, 고유 식별자 수집 방법과 DODownloadMode 권장 설정을 정리했다."
category: report
date: 2026-07-28
author: wychoi-orangelabs
tags: [Windows, 자산 식별, MachineGUID, Sysprep, Delivery Optimization, 콜센터, 보안, Orange Platform]
---

## 개요

한 고객사(콜센터 환경)의 Orange 관리 대상 PC를 분석하는 과정에서 확인된 두 가지 사안을 정리하고, 조사 방법과 해결 방향을 도출한다.

- **이슈 A** — 고객사 PC의 실제 Windows 컴퓨터 이름(hostname/NetBIOS)이 서로 중복 설정됨
- **이슈 B** — Windows 다운로드 최적화(Delivery Optimization, 전송 최적화) 설정 검토 및 최적화

**핵심 요약**: 두 사안은 서로 다른 계층의 문제지만 **마스터 이미지 복제 기반으로 동일 사양 단말을 다수 배포**라는 공통 원인에서 파생된다. DO(다운로드 최적화)의 피어 식별은 PC 이름이 아니라 **랜덤 GUID 기반**이므로 PC명 중복이 DO 오작동을 직접 유발하지는 않는다. 따라서 두 사안은 자산 식별 체계와 업데이트 배포 정책을 각각 정비하는 방향으로 접근한다.

## 1. 이슈 A — PC명(컴퓨터 이름) 중복

### 1.1 현상

고객사 다수 PC의 Windows 컴퓨터 이름이 동일하게 설정되어 있다.

### 1.2 추정 원인

- 마스터 이미지 복제(클론) 배포 시 `Sysprep /generalize` 미실행 → 컴퓨터 이름·Machine SID·MachineGUID가 그대로 복제
- 수동 재설치/재구축 시 동일한 명명 규칙을 재사용
- 참고: NetBIOS 컴퓨터 이름은 최대 15자 제한이 있어 규칙 기반 명명 시 앞부분이 잘려 서로 다른 자산이 같은 이름으로 수렴할 가능성도 있음

### 1.3 영향 (보안 / 관리 관점)

- 관리 도구가 PC명을 식별 키로 사용하는 경우 자산 충돌·오귀속 발생 → 수집 로그/사용자 활동을 특정 단말에 정확히 귀속시킬 수 없어, 포렌식·모니터링 제품의 근본 신뢰성이 훼손됨
- AD 도메인 조인 환경에서 컴퓨터 계정 충돌 가능
- SID/MachineGUID까지 동일 복제된 경우 식별·라이선스 기반 로직 오작동 가능

### 1.4 조사 방법 (각 PC에서 고유 식별자 수집·비교)

| 식별자 | 수집 명령 | 비고 |
|---|---|---|
| 컴퓨터 이름 | `hostname` | NetBIOS/DNS 이름 |
| MachineGUID | `Get-ItemPropertyValue -Path 'HKLM:\SOFTWARE\Microsoft\Cryptography' -Name MachineGuid` | OS 설치/복제 시 생성. Sysprep 미실행 시 복제됨 |
| Machine SID | `psgetsid` (Sysinternals) | 로컬 컴퓨터 SID (도메인 사용자 SID와 구분) |
| BIOS 시리얼 | `(Get-CimInstance Win32_BIOS).SerialNumber` | 하드웨어 고유값 |
| 메인보드/SMBIOS UUID | `(Get-CimInstance Win32_ComputerSystemProduct).UUID` | 하드웨어 고유값(가장 안정적) |
| NIC MAC | `Get-CimInstance Win32_NetworkAdapterConfiguration \| ? IPEnabled \| select MACAddress` | 물리 NIC 기준 |

→ 위 값을 종합해 (이름 / MachineGUID / SID / HW-UUID / MAC) **중복 매트릭스**를 작성하고, 어느 계층까지 중복되는지(단순 이름 중복 vs 이미지 통째 복제) 판별한다.

## 2. 이슈 B — 다운로드 최적화 (Delivery Optimization)

### 2.1 기능 개요

- 위치: 설정 > Windows Update > 고급 옵션 > 전송 최적화(다운로드 최적화)
- Windows Update, Microsoft Store 앱, 일부 MS 업데이트를 CDN(HTTP) + P2P 피어 캐시로 분산 다운로드하는 기능

### 2.2 핵심 동작 (Microsoft 공식 문서 기준)

다운로드 모드(`DODownloadMode`):

| 값 | 모드 | 설명 |
|---|---|---|
| 0 | HTTP 전용 | P2P 비활성. 원본/Connected Cache에서 HTTP만 |
| 1 | LAN (기본) | 동일 공인 IP(NAT) 뒤 단말끼리 피어 공유 |
| 2 | 그룹 | 동일 GroupID 단말끼리 피어링(서브넷 넘어 가능) |
| 3 | 인터넷 | 인터넷 피어 소스 허용 |
| 99 | 단순 | DO 클라우드 미접속(오프라인/폐쇄망), HTTP만 |
| 100 | 우회 | Windows 11에서 폐기(Deprecated) — 사용 금지, 0 또는 99로 대체 |

주요 사실:

- 피어 클라우드에서 단말은 PC 이름이 아니라 **"랜덤 생성 GUID"**로 식별된다 → PC명 중복은 DO 피어 식별에 직접 영향을 주지 않음
- 피어 매칭 기준: ContentID + GroupID + 외부 공인 IP/지리적 위치
- 콘텐츠 무결성: 피어에서 받은 조각마다 SHA-256 해시 검증, 불량 조각 폐기 및 악성 피어 차단. 개인 파일/폴더는 공유 대상이 아님
- P2P 통신 포트: TCP 7680

### 2.3 콜센터 환경 관점 분석 포인트

- **보안 관점**: 사내 PC 간 P2P 콘텐츠 공유가 발생. 공유 대상은 서명·해시 검증된 MS 콘텐츠에 한정되나, 고객정보를 취급하는 콜센터의 폐쇄망/망분리 정책상 P2P 트래픽 자체를 통제 대상으로 볼 수 있음
- **대역폭 관점**: 동일 사양 PC가 대량인 환경에서는 LAN/그룹 모드가 WAN 업데이트 대역폭 절감에 크게 유리
- **트레이드오프**: 보안(P2P 차단) ↔ 대역폭 절감(P2P 활용)

### 2.4 권장 설정 (환경 정책에 따라 택일)

- **보안/폐쇄망 우선**: `DODownloadMode = 99`(단순, 클라우드 미접속) 또는 `0`(HTTP 전용). `100`(우회)은 폐기되었으므로 사용하지 않음
- **사내 대역폭 절감 우선**: `DODownloadMode = 2`(그룹) + `DORestrictPeerSelectionBy`로 서브넷/mDNS 제한 → 사내 한정 피어링
- **VPN**: 기본적으로 VPN 감지 시 피어링 비활성(`DOAllowVPNPeerCaching` 기본 off)

### 2.5 조사 / 진단 방법 (PowerShell)

```powershell
# 현재 DO 상태/피어링 통계
Get-DeliveryOptimizationStatus
Get-DeliveryOptimizationPerfSnap

# DO 활동 로그 분석
Get-DeliveryOptimizationLog | Out-File do_log.txt

# 현재 적용 모드/정책 확인
Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\DeliveryOptimization\Config'
Get-ItemProperty 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\DeliveryOptimization'  # GPO/MDM로 강제된 값
```

→ 고객사 PC들의 현재 `DODownloadMode` 값과 정책 강제 여부를 수집해 실태를 파악한 뒤 §2.4 권장안과 비교한다.

## 3. 결론 — 두 사안의 관계

- DO의 피어 식별은 랜덤 GUID 기반이므로, PC명 중복이 DO 오작동을 직접 유발하지는 않는다.
- 다만 두 사안 모두 **"고객사 환경이 마스터 이미지 복제 기반의 동일 단말 다수로 구성"**이라는 공통 원인에서 파생된다. 따라서 ① 자산 식별 체계 정비(이슈 A)와 ② 업데이트/DO 배포 정책 정비(이슈 B)를 함께 진행한다.

## 4. 해결 방향

- **이슈 A**: 자산 식별 키를 PC명 단일 의존 → 안정적 고유 키 조합(HW-UUID + NIC MAC + 설치 시 Agent 발급 GUID)으로 전환. 신규 배포 시 `Sysprep /generalize` 표준화 및 명명 규칙 자동화, 기존 중복 단말 재명명 절차 수립.
- **이슈 B**: 환경 정책(보안 우선 vs 대역폭 우선)에 맞춰 `DODownloadMode`를 확정하고 GPO/MDM으로 고정.

## 참고 자료 (Microsoft 공식 문서)

- Delivery Optimization reference (DODownloadMode·GroupID 등)
- How Delivery Optimization works (피어 식별·무결성)
- Configure Delivery Optimization for Windows

*Orange Platform 고객사 환경 분석 리포트입니다.*
