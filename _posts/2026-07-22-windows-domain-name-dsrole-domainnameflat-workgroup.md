---
title: "워크그룹 PC의 도메인명이 빈 문자열로 나가는 이유 — DsRole의 DomainNameFlat을 버리고 서비스 의존 API로 재조회하던 결함"
excerpt: "에이전트가 인증 시마다 도메인명을 재조회하는데, 워크그룹 PC에서 값이 빈 문자열로 수집될 수 있는 경로를 역추적한 기록. DsRoleGetPrimaryDomainInformation이 이미 반환한 DomainNameFlat을 버리고, LanmanWorkstation 서비스에 의존하는 NetWkstaGetInfo 폴백으로 같은 값을 재조회하던 구조적 문제를 Windows API 실측으로 규명했다."
category: tech
date: 2026-07-22
author: kim-tigerj
tags: [Windows API, DsRoleGetPrimaryDomainInformation, NetWkstaGetInfo, 워크그룹, 도메인, 서비스 의존성, 부팅 레이스, Orange Platform]
---

## 배경

manager-web에 남아 있던 status2 System 병합 방어 로직의 유효성을 조사하던 중, 노드의 도메인명이 빈 문자열로 수집·전파될 수 있는 경로를 역추적한 결과 에이전트의 도메인명 취득 함수에서 근본 원인을 확인했다.

에이전트는 인증(`POST /v3/node`)할 때마다 GLOBAL 핸들러에서 도메인명을 재조회해 payload의 `System.DomainName`에 담는다. 인증은 부팅·절전 복귀·네트워크 변경·MQTT 재연결 때마다 일어나므로, **조회가 한 번 실패하면 그 시점의 `""`가 서버로 전파되어 DB의 정상 값을 덮어쓸 수 있다.**

## 원인

`GetDomainName`의 취득 구조는 다음과 같다.

- `DsRoleGetPrimaryDomainInformation`(lsass 로컬 조회, 서비스 기동 여부와 무관)을 호출하지만, 결과 구조체에서 `DomainNameDns`만 읽는다.
- 워크그룹 PC는 DNS 도메인이 없어 `DomainNameDns`가 NULL이다. 그런데 같은 구조체의 `DomainNameFlat`에 워크그룹 이름이 이미 들어 있음에도 이를 버린다. (MSDN: `MachineRole`이 Standalone인 경우 `DomainNameFlat`은 워크그룹 이름)
- 그 결과 워크그룹 PC는 전부 폴백인 `GetWorkGroupName` → `NetWkstaGetInfo`로 값을 얻는다. 이 API만 Workstation(LanmanWorkstation) 서비스 RPC에 의존하며, 서비스 미기동 시 `NERR_WkstaNotStarted`로 실패한다.
- yagent 서비스는 의존성 없이 등록되므로(`CreateService`의 `lpDependencies`가 NULL) 부팅 시 Workstation보다 먼저 실행될 수 있다. 이때 폴백이 실패하면 `""`가 나간다.

운영 노드 대다수(관측된 운영 환경 기준 약 94%)가 워크그룹 PC라, 사실상 전 노드가 **손에 쥔 값을 버리고 서비스 의존 경로로 같은 값을 다시 조회하는** 구조였다.

## 실측

로컬 워크그룹 PC에서 두 API를 직접 호출해 비교한 결과:

```
DsRoleGetPrimaryDomainInformation: ret=0, MachineRole=0(StandaloneWorkstation)
  DomainNameFlat = [WORKGROUP]     ← 첫 호출이 이미 워크그룹 이름 반환
  DomainNameDns  = []              ← 현재 코드는 이것만 읽고 버림
NetWkstaGetInfo(102): ret=0
  wki102_langroup = [WORKGROUP]    ← 폴백이 재조회하는 값. DomainNameFlat과 동일
```

발생 빈도 참고: 운영 노드 현재 상태와 로컬 에이전트 에러 로그에서 빈 `DomainName` 발생 흔적은 0건이다. 경로는 구조적으로 실재하나 관측된 바는 없다.

## 수정

`DomainNameDns`가 비어 있으면 `DomainNameFlat`을 사용한다.

기존:

```cpp
if (ERROR_SUCCESS == (dwRet = pDSROLE_PRIMARY_DOMAIN_INFO_BASIC(NULL, DsRolePrimaryDomainInfoBasic, (PBYTE*)&pInfo)))
{
    ::StringCbCopy(pName, dwSize, pInfo->DomainNameDns ? pInfo->DomainNameDns : _T(""));
    pDsRoleFreeMemory(pInfo);
}
```

변경:

```cpp
if (ERROR_SUCCESS == (dwRet = pDSROLE_PRIMARY_DOMAIN_INFO_BASIC(NULL, DsRolePrimaryDomainInfoBasic, (PBYTE*)&pInfo)))
{
    //  도메인 가입 PC는 DomainNameDns(DNS 이름)가 채워지고,
    //  워크그룹 PC는 DomainNameDns가 NULL, DomainNameFlat이 워크그룹 이름이다.
    //  둘 다 lsass 로컬 조회라 LanmanWorkstation 서비스 기동 여부와 무관.
    PCWSTR pDomain = (pInfo->DomainNameDns && *pInfo->DomainNameDns)
                        ? pInfo->DomainNameDns : pInfo->DomainNameFlat;
    ::StringCbCopy(pName, dwSize, pDomain ? pDomain : _T(""));
    pDsRoleFreeMemory(pInfo);
}
```

- 도메인 가입 PC: 지금과 동일하게 DNS 이름(예: `corp.example.com`). 표기 변화 없음.
- 워크그룹 PC: 지금과 동일한 값(`WORKGROUP` 등)을 서비스 의존 없이 취득. 실측으로 두 API의 값이 동일함을 확인.
- `GetWorkGroupName` 폴백은 DsRole 자체가 실패하는 경우의 최후 보험으로 그대로 유지한다.

## 영향 범위

- 호출처: `GetDomainName`은 인증 payload(GLOBAL 핸들러)와 초기 GLOBAL 구성(`SetGLOBAL`)에서 사용된다. 모든 호출처가 동일한 개선 효과를 받는다.
- 서버·매니저 웹 계약 변화 없음. 값의 의미와 형식이 동일하다.

## 검증 포인트

- 워크그룹 PC에서 인증 payload의 `System.DomainName`이 기존과 동일한 워크그룹 이름인지
- 도메인 가입 PC에서 DNS 도메인 이름이 기존과 동일하게 나오는지
- 도메인 가입·탈퇴 전환 시 다음 인증에서 새 값이 반영되는지

---

*이 글은 Orange Platform 에이전트의 Windows 도메인명 취득 로직 분석 및 트러블슈팅 리포트입니다.*
