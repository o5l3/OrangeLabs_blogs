---
title: "코드 서명 판정에 신뢰 결과 반영하기 — 서명자 이름만 믿으면 변조 서명이 게이트를 통과한다"
excerpt: "Authenticode 검증이 서명자 CN만 파싱하고 신뢰 판정(VERIFY_RESULT)을 버리면 만료·폐기·변조 서명도 통과한다. 폐쇄망 제약 하에서 다이제스트 불일치(변조)만 오프라인으로 판별해 신뢰 판정을 생산·보관한 보안 수정 기록."
category: tech
date: 2026-08-07
author: kim-tigerj
tags: [코드서명, Authenticode, 보안, 변조탐지, Windows에이전트, Orange Platform]
---

## 문제

`CCodeSign`의 코드 서명 판정이 **서명자 문자열(CN)만 파싱하고 신뢰 판정 결과(VERIFY_RESULT)를 버린다.** `PhVerifyFile`은 만료·폐기·미신뢰 루트·다이제스트 불일치(변조)여도 서명 블롭이 있으면 서명자 이름을 반환하므로, "서명자에 microsoft 포함" 같은 게이트는 **신뢰되지 않은 서명도 통과**시킨다.

## 현황 (수정 전)

- `yagent21/Nt2.cpp` `PhVerifyFile`: `numberOfSignatures != 0`이면 서명자 이름을 채움. 실제 신뢰 판정은 반환값 `verifyResult`(`VrTrusted`/`VrExpired`/`VrRevoked`/`VrDistrust`/`VrBadSignature`)에 담김.
- `yagent21/CCodeSign.h` `_VerifyFile`은 `vr`를 받아 반환하지만, `CreateCodeSign`·`GetCodeSignDirectly`가 `vr`를 버리고 cert 문자열의 CN만 파싱.
- `CODE_SIGN` 구조체에 판정 필드가 없어 캐시 경로(`GetCodeSignWithCache`)에서 신뢰 판정이 완전히 소실됨.

## 제약 (폐쇄망)

대부분 고객이 폐쇄망이라 온라인 신뢰 검증(CRL/OCSP 폐기 확인, 타임스탬프 조회)이 불가능하다. `VrTrusted` 전면 요구는 정상 MS DLL도 미신뢰로 떨어뜨려 오작동한다. 따라서 신뢰 사슬 전체 검증은 채택하지 않고, **`VrBadSignature`(다이제스트 불일치=변조)만 판별**한다.

`VrBadSignature`는 네트워크 없이 파일 해시만으로 판정되는 **오프라인 확실값**이다. `TRUST_E_BAD_DIGEST` 하나만 이 값으로 매핑되고(`Nt2.cpp` `PhpStatusToVerifyResult`), 그 외 오류는 전부 `VrSecuritySettings`로 가므로 오탐 위험이 낮다. `PhVerifyFile`은 이미 `PH_VERIFY_PREVENT_NETWORK_ACCESS`로 검증한다.

**적용 범위**: `VrBadSignature`는 내장(embedded) 서명 파일의 변조·손상에서만 발생한다. 카탈로그 서명 파일이 변조되면 해시가 카탈로그에서 안 찾아져 `VrNoSignature`로 나오고, 이 경우 서명자 이름 자체가 안 채워져 기존에도 게이트를 통과하지 못한다. 즉 이번 수정이 새로 막는 것은 **내장 서명 파일의 변조·손상**이다.

## 적용 내역 (`yagent21/CCodeSign.h`)

- `CODE_SIGN`에 `bool bVerify` 추가 — 의미: 변조(`VrBadSignature`) 검출 안 됨. 무서명·검증불가도 `true`이며 서명 존재 여부는 `signer`로 판단한다. 생성자에서 `false` 초기화. 파생(`CODE_SIGN2`)이 아닌 베이스에 둬서 캐시 출력 복사(베이스 슬라이스 대입)에서도 전파된다.
- 생산 지점 2곳(`GetCodeSignDirectly`, `CreateCodeSign`)에서 `_VerifyFile` 직후 `bVerify = (VrBadSignature != vr)` 저장. 서명자 문자열은 기존대로 유지(정보 표시 용도) — 판정은 소비처가 `bVerify`로 한다.
- **`GetCodeSignDirectly` 진입부 방어** — nullptr 가드 + `signerUID`·`bVerify`·`signer`·`value` 리셋. 재사용 객체에 이전 파일의 서명 문자열이 잔존해, 무서명 파일이 이전 파일의 서명자로 파싱되던 경로를 함수 안에서 봉쇄.
- **`CreateCodeSign` 예외 시 `return nullptr`** — 예외가 나면 미완성(빈 signer·value) 객체가 캐시에 영구 등록되고 성공으로 반환되던 결함 수정. 캐시에 남지 않으므로 다음 호출에서 재시도된다.
- **`GetCodeSignWithCache` 캐시 무효화** — 캐시 히트여도 `nSize`가 다르면 엔트리를 버리고 재검증. 같은 경로에 파일이 교체되면 이전 파일의 서명 정보가 반환되던 문제 수정. 크기까지 같은 교체는 미검출(수용).

## 범위 제외 / 후속

- 소비처의 `bVerify` 검사(서명 게이트에서 `bVerify=false` 배제)는 별도 이슈로 진행. 이번 이슈는 신뢰 판정의 생산·보관까지.
- 프로세스 문서(JSON)의 Signer 게이트는 filelist DB를 거쳐 온 문자열이라 `bVerify`가 전달되지 않음 — 소비처 이슈에서 함께 검토.

## 조사 중 발견한 기존 결함 (보고만)

- `CFileList.Thread.cpp:227` — `FastHash`가 bool로 붕괴(`asUInt()? true : false`)되어 이후 `FastHash == FastHash2` 비교가 사실상 항상 거짓. 파일 불변 검증 분기 전체가 도달 불가. 이 앞단 버그 덕에 루프 바깥 `CODE_SIGN` 재사용 오염이 현재는 발현하지 않았고, 위 진입 리셋으로 선제 봉쇄됨.
- `IsWindowsOsService`의 서비스명 캐시는 일시 실패 판정도 영구 유지(무효화·재시도 없음).

---

*Orange Platform 엔드포인트 에이전트의 코드 서명 신뢰 판정 보안 리포트입니다.*
