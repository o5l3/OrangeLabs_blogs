---
title: "DNS ETW 이벤트 ID의 의미를 잘못 읽으면 — 탐지 6.6배 폭증과 죽어 있던 실패 탐지"
excerpt: "에이전트의 DNS-Client ETW 처리에서 이벤트 ID의 실제 의미가 어긋나 있었다. 로컬 캐시 조회를 서버 통신으로 착각해 정상 PC에서 탐지가 2,491건→13,509건으로 뛰었고, 정작 진짜 실패 탐지(3008)는 통째로 죽어 있었다. 매니페스트와 폐쇄망 재현으로 이벤트 하나하나의 의미를 확정한 기록."
category: tech
date: 2026-08-19
author: wychoi-orangelabs
tags: [ETW, DNS, DNS-Client, 이벤트추적, 탐지룰, Windows, Orange Platform]
---

## 개요

에이전트의 DNS ETW 이벤트 처리에서, 이벤트 ID와 실제 의미가 어긋나 있었고 처리 로직이 그 오해 위에 세워져 있었다.

로컬 DNS 캐시 조회 한 쌍(3016·3018)을 서버 통신 이벤트로 착각해, 캐시 미스를 "DNS Server Error"로, 캐시 조회 시작을 "DNS Timeout"으로 보고했다. 추적 과정에서 **3008 실패 탐지가 통째로 죽어 있던 것**도 함께 드러났다.

## 왜 지금 드러났나

두 case(3016/3018)는 원래 죽은 코드였다. 상태 필드를 읽는 함수가 `QueryStatus`만 읽는데 3016/3018은 `Status`로 값을 싣기 때문이다. 별건에서 DNS 캐시 결함을 추적하다 `Status` 폴백을 넣었고, 그러면서 두 case가 처음으로 살아나 잘못된 처리가 노출됐다.

## 영향

정상 동작하는 PC에서 탐지가 **2,491건 → 13,509건**으로 뛰었다. 그중 **8,274건(61%)**이 case 3018의 캐시 미스 한 경로에서 나왔다. 정상적으로 잘 돌아가는 PC가 하루 종일 "DNS 서버 오류"를 쏟아내는 상태였다.

## 근거는 매니페스트다

MS 공식 문서에는 이 이벤트들의 레퍼런스가 없다. 근거는 **매니페스트**이며 한 줄로 재현된다.

```
wevtutil gp Microsoft-Windows-DNS-Client /ge /gm:true > dns_manifest.txt
```

상태 코드는 SDK의 `winerror.h`가 유일한 근거다(10.0.28000.0 전수 대조). 한 이벤트 ID에 버전 0·1·2가 공존하며, 뒤 버전일수록 ClientPID·QueryBlob이 붙는다.

## 조기 return 목록 확정

상태 필드가 없거나, 있어도 질의 결과에 영향을 주지 않는 **진행 단계** 이벤트를 걸러냈다.

| 이벤트 | 제외 근거 |
|---|---|
| 1015 | 재시도마다 나는 중간 결과. 최종 판정은 1013이 한다 |
| 1022 | 폴백 안 함 통지. 실패가 아니다 |
| 3006 / 3009 / 3010 / 3019 | 질의 진행 단계. 결과가 없다 |
| 3016 | 로컬 캐시 조회 시작. 서버와 통신하지 않는다 |
| 3018 | 로컬 캐시 조회 결과 |
| 3020 | 인터페이스별로 발생해 정상망에서도 타임아웃을 낸다 |

## 3008 실패 탐지가 통째로 죽어 있었다

이번 점검에서 발견한 가장 큰 결함이다. 탐지 조건에 적힌 네 코드가 상태 문자열 변환 함수에 하나도 등록되어 있지 않았다.

```cpp
(3008 == eventId && (
    ERROR_INVALID_NAME == queryStatus     ||   // 123    미등록
    ERROR_OBJECT_NOT_FOUND == queryStatus ||   // 4312   미등록
    WSAEADDRNOTAVAIL == queryStatus       ||   // 10049  미등록
    WSAENOTCONN == queryStatus))               // 10057  미등록
```

그러면 이렇게 흘러간다.

```cpp
json["StatusString"] = FormatDnsQueryStatus(123);   // -> ""  (nullptr -> 빈 문자열)
...
if (PID && statusString.length()) {                 // -> 0 이라 거짓
```

탐지가 만들어지지 않고 조용히 버려진다. 반면 억제 카운터는 올라간다. **탐지는 못 하면서 SUSPEND는 걸리는** 최악의 조합이다. 지금까지 안 드러난 이유는, 탐지 대부분이 3018(9701 등록됨)과 3020에서 나오고 있었기 때문이다. 그 둘을 조기 return시키자 3008만 남았고, 그 3008이 사실 안 돌고 있었다.

수정은 둘이다. 코드 6개(123, 4312, 10049, 10051, 10054, 10057)를 등록했고, 재발 방지를 위해 폴백을 넣었다.

```cpp
//  미등록 코드도 숫자로 남긴다. 빈 문자열을 돌려주면
//  switch 뒤의 statusString.length() 검사에서 탐지가 조용히 사라진다.
char buffer[32] = { 0 };
sprintf_s(buffer, "STATUS_%u", status);
return buffer;
```

## 타임아웃·서버 오류 탐지 도입

기존엔 3008 하나뿐이던 실패 탐지에 둘을 추가했다.

**1013 / 1014 — 진짜 이름 해석 타임아웃.** 상태 필드가 없어 관문을 통과할 수 없다. `IsFailureEventId()`로 갈라내 우회시킨다.

```cpp
const bool isFailureEventId = IsFailureEventId((USHORT)eventId);
if (!isFailureEventId && !GetQueryStatusFromEvent(EventRecord, queryStatus)) {
    return;
}
```

상태 필드가 없으므로 StatusString을 직접 `"DNS_NAME_RESOLUTION_TIMEOUT"`으로 채운다. 비워두면 뒤의 `statusString.length()` 검사에 걸려 버려진다.

**3011 — 서버가 장애 RCODE로 답한 경우만.** 응답을 받았다는 것 자체가 서버가 살아 있다는 증거다. NXDOMAIN·NO_RECORDS는 정상적인 부정 답변이므로 제외하고, SERVER_FAILURE·REFUSED·FORMAT_ERROR만 남긴다.

## 폐쇄망 재현 방법

이 판정들을 검증하려면 실제로 DNS를 막아봐야 한다.

```powershell
wevtutil sl Microsoft-Windows-DNS-Client/Operational /e:true

New-NetFirewallRule -DisplayName "DNS-BLACKHOLE-UDP" -Direction Outbound `
    -Protocol UDP -RemotePort 53 -Action Block -Profile Any
New-NetFirewallRule -DisplayName "DNS-BLACKHOLE-TCP" -Direction Outbound `
    -Protocol TCP -RemotePort 53 -Action Block -Profile Any

ipconfig /flushdns
Resolve-DnsName blackhole-a1.example.com -Type A -DnsOnly

Remove-NetFirewallRule -DisplayName "DNS-BLACKHOLE-*"
```

핵심 주의 사항이 여럿 있다.

- **드롭이어야 한다.** 방화벽 Block은 조용히 드롭해 타임아웃을 만든다. ICMP unreachable을 돌려주는 잘못된 IP로는 거부가 되어 타임아웃이 안 난다.
- **nslookup은 쓰면 안 된다.** 자체 리졸버라 DNS Client 이벤트를 안 낸다. `Resolve-DnsName` / `[System.Net.Dns]::GetHostAddresses()`를 쓴다.
- **DoH 확인.** `Get-DnsClientDohServerAddress`로 Auto-upgrade가 꺼져 있는지 본다. 켜져 있으면 443으로 새서 53 차단이 무의미해진다.
- 매 시도 전 `ipconfig /flushdns`. 부정 캐시로 끝나면 서버까지 안 간다.

`Microsoft-Windows-DNS-Client/Operational` 채널은 기본 꺼져 있으며, 켜면 에이전트와 무관하게 독립 검증이 된다.

## 3018을 조기 return하는 근거

3018 상태 분포 (수집본 전체 37,871건)

| 상태 | 건수 | 뜻 |
|---|---|---|
| RECORD_DOES_NOT_EXIST(9701) | 28,996 | 캐시 미스. 정상 |
| 성공(0) | 8,499 | 캐시 히트. 정상 |
| NO_RECORDS / NXDOMAIN / TIMEOUT / SERVFAIL | 376 (1.0%) | 부정 캐시 |

남는 1.0%는 이전 실패를 캐시가 기억했다 되돌려준 메아리다. 3018이 부정 캐시 상태를 낸 도메인 9개는 **3008에서도 100% 실패가 관측**됐다. 즉 3018을 죽여도 놓치는 실패가 없다.

## 3008 ERROR_TIMEOUT과 1013은 완전 중복이다

검증 구간 대조 결과다.

```
1013                12건 / 고유 이름 10개
3008 ERROR_TIMEOUT  25건 / 고유 이름 10개

이름 교집합       10 / 10
1013 에만            0
3008 에만            0
```

1013을 남기는 이유는 셋이다. 질의당 1회다(3008은 QueryType A/AAAA별로 발생해 2배가 된다). "설정된 DNS 서버가 모두 무응답"으로 원인이 확정된다(3008의 1460은 LLMNR 멀티캐스트 무응답으로도 난다). 그리고 Address에 서버 주소를 싣는다(3008에는 서버 필드가 없다).

## 1015가 실제로 발생하는지 — 재현 결과

한 이름의 재시도 사슬을 그대로 관측했다.

```
19:53:43.045  3010   1차 전송
19:53:44.048  1015   1차 무응답 (1초 후)
19:53:45.060  1015   2차 무응답 (1초 후)
19:53:47.070  1015   3차 무응답 (2초 후)
19:53:51.084  1015   4차 무응답 (4초 후)
19:53:55.099  1015   5차 무응답 (4초 후)
19:53:55.100  1013   최종 포기
19:53:55.100  3008   ERROR_TIMEOUT 반환
```

3010과 1015가 1:1로 짝을 이룬다. 1015는 "이번 시도에 이 서버가 무응답"(서버×재시도 단위, Information 레벨), 1013은 "끝내 어느 서버도 응답 안 함"(질의 1회, Error 레벨)이다. 억제 카운터 100 기준으로 보면 1013만으로는 실패 도메인 100개가 있어야 도달하지만, 1013+1015로 보면 약 15개면 도달한다.

## 수정 결과 확인

전체 빌드 후 3차 재현(435건)에서 조기 return 402건(86%), 1013 탐지 10건, 3008 실패 탐지(status=123) 4건이 확인됐다. status=123 4건이 핵심이다. `ERROR_INVALID_NAME`은 그동안 StatusString이 비어 통째로 사라지던 것인데, 이제 탐지 대상으로 잡힌다.

수정 단계별 누적 효과(2차 검증 구간 817건 기준)

| 단계 | 탐지 |
|---|---|
| 1015까지 탐지했다면 | 116 |
| 1015 조기 return | 37 |
| 3008 ERROR_TIMEOUT 중복 제거 | 12 |
| 상태 코드 등록 후 | 약 21 |

이벤트 하나하나의 의미를 매니페스트로 확정하고 나니, 정상 PC의 소음을 6.6배 줄이면서 동시에 그동안 놓치고 있던 진짜 실패를 잡게 됐다. ETW 이벤트는 이름만 보고 의미를 짐작하면 안 되고, 매니페스트와 실측으로 확인해야 한다는 것이 이 작업의 교훈이다.

*Orange Platform 에이전트의 DNS ETW 이벤트 처리 정정에 관한 기술 리포트입니다.*
