---
title: "정상 트래픽이 탐지를 지우고, 정상 RST가 탐지를 채운다 — ETW TCP 노이즈 처리 두 가지"
excerpt: "에이전트의 ETW TCP 처리에서 벤더 노이즈 필터가 서킷 브레이커·TCB 정리보다 앞서 리턴해 카운터와 연결 맵을 망가뜨리고 있었다. 서브스트링 매칭은 무관 도메인의 증상을 조용히 숨겼고, 공유 IP는 마지막 DNS 결과에 따라 필터 여부가 뒤집혔다. 필터를 장부 처리 뒤로 옮기고 경계 매칭으로 바꾼 것, 그리고 정상 운영에서도 반복되는 TCP RST 탐지에 시간창 레이트 리밋을 건 기록."
category: tech
date: 2026-08-28
author: kim-tigerj
tags: [ETW, TCP, 네트워크, 탐지룰, 노이즈필터, 레이트리밋, Windows, Orange Platform]
---

## 개요

에이전트의 ETW TCP 이벤트 처리에서 노이즈 필터가 잘못된 위치에 있었다. 벤더 트래픽(chrome·msedge 프로세스, MS·Google 도메인)을 걸러내는 조기 리턴이 서킷 브레이커와 TCB 맵 정리보다 앞서 있어, 필터 대상 트래픽이 통계와 연결 맵에서 통째로 빠지고 있었다. 여기에 서브스트링 매칭·공유 IP 처리·로그 볼륨 문제를 함께 정리했다. 뒤이어, 정상 운영에서도 반복되는 TCP RST가 탐지를 채우는 문제에 시간창 레이트 리밋을 걸었다.

## 필터 위치 이동 — 카운터와 연결 맵을 되살린다

필터 조기 리턴(`CETW.TCP.cpp:794-825`)이 두 가지보다 앞에 있었다.

**서킷 브레이커(`:827~`)** — 필터 대상 트래픽의 1033 성공 리셋과 1034 실패 증가가 모두 카운터에서 빠진다. 사무 PC에서는 브라우저가 주 트래픽이라 복구 신호(1033 리셋)의 본체가 사라지고, 프록시 장애처럼 MS·Google 방향만 실패하는 상황에서는 폭주 차단기가 아예 안 걸린다.

**TCB 맵 정리(1033 `:956`, 1034 `:996`, 1040 `:1033`, 1043 `:1062`, 1045 `:1084`)** — 1002 시점엔 캐시 미스로 통과했다가 그 직후 DNS 응답이 캐시에 적재되면, 완료 이벤트만 필터에 걸려 `g_tcpConnections` 엔트리가 정리되지 않고 잔류한다. 연결과 DNS 해석이 거의 동시인 첫 접속에서 자연스럽게 생기는 순서다. stale 정리는 새 1002 때만 돈다.

수정은 세 필터의 조기 리턴을 상단에서 빼서, 이벤트별 switch가 끝난 뒤 job 생성 직전으로 옮기는 것이다.

```
현재 순서:
ip·domain·bFiltered 추출
[필터 3종 → return]        ← 문제 위치
서킷 브레이커 (1033 리셋 / 1034 증가)
ExtractAllFields
이벤트별 switch (TCB 맵 삽입·삭제, info 구성)
Domain 관문 → IsSkipDomain → job 생성

바꿀 순서:
ip·domain·bFiltered 추출    (유지 — switch의 info["Domain"]이 쓴다)
서킷 브레이커              (필터 트래픽도 카운터 반영 = 머지 이전 동작 복원)
ExtractAllFields
이벤트별 switch            (TCB 정리까지 전부 수행)
[필터 3종 → return]          ← 이동 위치: bookkeeping 끝, job 생성 직전
Domain 관문 → IsSkipDomain → job 생성
```

이동 코드는 세 블록을 한 조건으로 합치면 된다. switch 닫는 `}` 다음, Rule 매칭 처리 관문(`:1263`) 앞이다.

```cpp
//  노이즈 필터 — 카운터 집계·TCB 맵 정리까지 마친 뒤, 탐지(job) 생성만 차단한다.
if (bFiltered || IsSkipByProcess(tpptr) || IsSkipByIp(ip)) {
    return;
}
```

서킷 브레이커는 머지 이전 동작으로 복원되고, TCB 잔류가 없어진다. 필터된 이벤트도 `ExtractAllFields`는 타지만 머지 이전과 동일 수준이라 회귀가 아니다. 필터 목적(벤더 노이즈의 탐지·전송 차단)은 그대로 달성한다.

## 경계 매칭 전환 — 무관 도메인이 숨지 않게

기존(`CETW.DNS.cpp:166`)은 경계 검사 없는 `find()` 서브스트링 매칭이었다. `"windows"`·`"office"`·`"azure"`·`"bing"`·`"live.com"` 같은 일반 단어가 목록에 있어, `tubing.com`(bing)·`olive.com`(live.com)·그룹웨어류 도메인(office 포함) 등 무관 도메인의 TCP 증상이 조용히 숨었다.

수정은 단어를 레이블 단위 정확 일치로, 도메인 꼬리(`live.com` 등)를 점 경계 접미사 매칭으로 분리하는 것이다.

```cpp
static bool IsFilteredHost(const std::string& host)
{
    if (host.empty()) {
        return false;
    }

    std::string lowerHost = host;
    std::transform(lowerHost.begin(), lowerHost.end(), lowerHost.begin(),
        [](UCHAR c){ return (CHAR)::tolower(c); });

    //  레이블 단위 정확 일치만 매칭한다. 서브스트링 매칭은 무관 도메인을 숨긴다.
    static const char* filteredLabels[] = {
        "microsoft", "msidentity", "msft", "msn", "skype", "outlook",
        "onedrive", "sharepoint", "1drv", "bing", "msedge", "windows",
        "office", "azure", "github", "copilot", "visualstudio",
        "google", "chrome", "gstatic", "gvt1", "gvt2", "gvt3", "nvidia",
    };

    //  도메인 꼬리로만 매칭할 항목. 반드시 '.' 경계를 확인한다.
    static const char* filteredSuffixes[] = {
        "live.com", "pki.goog", "aka.ms", "sfx.ms",
    };

    size_t pos = 0;
    while (pos <= lowerHost.size()) {
        size_t end = lowerHost.find('.', pos);
        if (end == std::string::npos) end = lowerHost.size();
        const std::string label = lowerHost.substr(pos, end - pos);
        for (const char* w : filteredLabels) {
            if (label == w) {
                return true;
            }
        }
        pos = end + 1;
    }

    for (const char* s : filteredSuffixes) {
        const size_t sl = strlen(s);
        if (lowerHost.size() == sl && lowerHost == s) {
            return true;
        }
        if (lowerHost.size() > sl &&
            0 == lowerHost.compare(lowerHost.size() - sl, sl, s) &&
            '.' == lowerHost[lowerHost.size() - sl - 1]) {
            return true;
        }
    }

    return false;
}
```

주의할 점이 있다. 서브스트링에서 경계 매칭으로 바꾸면 매칭 범위가 줄어드는 항목이 있다. 예를 들어 `windowsupdate.com`은 원 코드 주석에서 제외 의도였으나 서브스트링 `"windows"`가 사실상 포함시키고 있었다. 전환 후 필터 히트 목록을 재관측해 목록을 확정한다.

## 공유 IP — last-writer-wins 완화

기존(`CETW.Network.h:134`)은 Insert가 같은 IP의 domain과 isFiltered를 마지막 해석 값으로 덮었다. CDN·공유 IP(Akamai·Cloudflare·구글 프론트엔드)는 하나의 IP를 수많은 도메인이 공유하므로, 마지막 DNS 결과에 따라 도메인 표기와 필터 여부가 함께 뒤집힌다. 벤더 도메인이 마지막이면 같은 IP로 붙는 고객 서비스 증상까지 삭제된다.

권장안은 엔트리에 단일 `isFiltered` 대신 최근 도메인 목록을 보관하고, 전원이 필터 대상일 때만 필터하는 것이다.

```cpp
struct DnsCacheEntry {
    std::string ip;
    std::vector<std::string> domains;   //  최근 도메인, 최대 4개(오래된 것부터 교체)
};
//  필터 판정: domains 전원이 IsFilteredHost()일 때만 true — 하나라도 일반 도메인이면 증상을 살린다.
//  표시: 가장 최근 도메인.
```

공유 IP에서 완전한 귀속은 구조적으로 불가능하므로, 원칙은 "필터로 증상을 지울 때는 보수적으로"다.

## IPv4 전용 명시

한 실측 함대 전체의 TCP 증상 91건 중 IPv6 주소는 0건이었다(2026-08-19 기준). 국내 기업 사내망은 IPv4 전용이 지배적이라 현재 고객 환경에서 IPv6 경로의 필터 우회는 실해가 없다. 캐시와 `IsSkipByIp` 목록이 IPv4 전용임을 주석으로 명시하고, IPv6 사용망 고객이 생기면 그때 별도로 확장한다.

## 로그 정리와 추출 실패 폴백

캐시 미스마다 찍히던 `"TCP domain == ip"` info 로그(`:785`)는 재시작 직후·DNS를 안 거친 연결(브라우저 자체 DoH, IP 직접 접속)·LRU 축출분에서 상시 발생해 하루 수만 줄이 쌓였다. 이 로그는 삭제하고, 필터 로그는 이동으로 통합된 한 곳에서 debug로 강등했다.

추출 실패 폴백도 복원했다. `RemoteAddress` 추출이 실패하면 빈 문자열 → `domain=""` → `IsSkipDomain("")=true`로 증상이 조용히 드롭됐다. 이벤트에서 주소를 못 뽑으면 연결 시작(1002) 때 저장한 TCB 값으로 폴백한다.

```cpp
//  이벤트에서 주소를 못 뽑으면 연결 시작(1002) 때 저장한 값으로 폴백한다.
if (remoteAddress.empty() && 0 != tcb) {
    std::lock_guard<std::mutex> lock(g_tcpMutex);
    auto it = g_tcpConnections.find(tcb);
    if (it != g_tcpConnections.end()) {
        remoteAddress = it->second.remoteAddr;
    }
}
```

## TCP RST 탐지 레이트 리밋

위 필터 정리에 이어, 1044(비정상 종료)를 신규 채택해 만든 "TCP Connection Reset" 탐지가 실제로 발화한다는 것이 확인됐다. `Rule/0008/TCP.json:436`에 `{ev.Status} $eq STATUS_CONNECTION_RESET` 활성 룰이 실존하기 때문이다. 문제는 1044에 폭주 방지가 없다는 것이다.

RST는 정상 운영에서도 반복된다 — 서버 keep-alive 정리, 중간 장비, 앱의 종료 방식. chrome·msedge는 프로세스 필터로 걸러지지만 사내 업무 앱·런처·업데이터의 RST는 남는다. 기존 서킷 브레이커(1034 카운트, 1033 성공 시 리셋)로는 못 막는다. 정상 RST 노이즈는 성공과 공존하므로 리셋이 계속 일어나 브레이커가 영원히 안 걸린다. 폭주 형태가 아니라 상시 배경 소음 형태라서, 브레이커가 아니라 시간창 레이트 리밋이 맞는 도구다.

창당 상한까지는 그대로 탐지해 진짜 증상을 보존하고, 초과분은 통계·TCB 장부만 남기고 탐지를 보류한다. 억제 방식은 기존 SUSPEND 관례와 같은 결이다.

```cpp
//  1044(RST) 탐지 레이트 리밋. RST 는 정상 운영에서도 반복된다(서버 keep-alive 정리, 중간 장비).
static const double RST_DETECT_WINDOW_MS      = 10 * 60 * 1000.0;   //  10분 창
static const DWORD  RST_DETECT_MAX_PER_WINDOW = 5;                  //  창당 탐지 상한
```

`case 1044`에서 Status 검사 3개를 지난 뒤, info 구성 앞에 게이트를 넣는다.

```cpp
case 1044:
{
    const std::string status = json.get("Status", "").asString();

    //  Status 0 으로도 온다. 끊긴 사유를 담지 않았으므로 증상이 아니다.
    if ("STATUS_SUCCESS" == status) {
        break;
    }
    //  정상 종료 계열은 증상이 아니다.
    if (IsNormalDisconnectStatus(status)) {
        break;
    }
    if ("STATUS_CONNECTION_REFUSED" == status ||
        "STATUS_IO_TIMEOUT"         == status ||
        "STATUS_TIMEOUT"            == status) {
        break;
    }

    //  STATUS_CONNECTION_RESET 활성 룰이 이 이벤트로 발화한다.
    //  정상 RST 는 성공과 공존해 기존 브레이커(성공 리셋)로는 못 막으므로 시간창으로 제한한다.
    const ULONGLONG now = GetCurrentTick();
    if (0 == dwRstWindowStart || TickToMs(now - dwRstWindowStart) > RST_DETECT_WINDOW_MS) {
        dwRstWindowStart = now;
        dwRstWindowCount = 0;
    }
    if (++dwRstWindowCount > RST_DETECT_MAX_PER_WINDOW) {
        if (RST_DETECT_MAX_PER_WINDOW + 1 == dwRstWindowCount) {
            GetDebugLogger()->info("---- SUSPEND TCP RST detect for window");
        }
        break;
    }

    info["Event"]     = "TCP Connection Reset";
    info["ProcessId"] = (Json::UInt)PID;
    info["Domain"]    = domain;
    json["Info"]      = info;
    AddTcpStats(json, false);

    bShouldProcess = true;
    break;
}
```

상수(10분·5건)는 제안값이며 운영 관측 후 조정한다. 검증은 RST 다발 환경에서 탐지가 창당 상한에서 멈추는지, 초과 시 debug 로그 1줄이 남는지, 상한 이내의 진짜 RST 증상은 기존대로 탐지되는지, 창 경과 후 카운터가 리셋되어 탐지가 재개되는지로 확인한다.

## 정리

TCP 노이즈 처리에서 반복되는 원칙은 하나다. 필터로 증상을 지울 때는 보수적으로, 그리고 장부(카운터·연결 맵)는 필터와 무관하게 끝까지 유지한다. 필터를 장부 처리 앞에 두면 정상 트래픽이 탐지 인프라를 갉아먹고, 서브스트링·last-writer-wins처럼 경계가 없는 매칭은 무관한 증상을 조용히 숨긴다. RST처럼 정상과 증상이 같은 신호를 공유하는 경우엔 폭주 차단기가 아니라 시간창 레이트 리밋이 맞는 도구다.

*Orange Platform 에이전트의 ETW TCP 노이즈 필터 보완과 RST 탐지 레이트 리밋에 관한 기술 리포트입니다.*
