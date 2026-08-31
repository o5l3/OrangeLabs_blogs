---
title: "무한 누적되는 시계열에 TTL을 걸기 위해 — 최초 관측·신원을 차원 테이블로 분리"
excerpt: "프로세스 텔레메트리 raw가 37일 만에 4.52GB로 무한 누적되는데 TTL을 걸 수 없었다. '이 프로그램을 처음 본 게 언제냐' 류의 평생 질문이 raw 전 기간 조회에 의존했기 때문이다. SUID당 1문서의 차원 테이블 sprocess_registry로 최초 관측·신원 메타를 이관해, 카드가 하나도 없어도 모든 응답이 정답이 되도록 만든 설계 판단 네 가지."
category: tech
date: 2026-08-28
author: smahn9123
tags: [MongoDB, 시계열, 데이터모델링, TTL, 백엔드, 마이그레이션, Orange Platform]
---

## 개요

프로세스 텔레메트리 raw(`sprocess`)의 "평생 질문"(최초 관측 시점·신원 메타)을 SUID당 1문서의 차원 테이블 `sprocess_registry`로 이관해, 후속 raw TTL의 전제를 만들었다.

raw는 무한 누적 중이었다. 한 실측 환경에서 실 agent 21대가 37일 만에 4.52GB·182만 doc를 쌓았다. 그런데 TTL을 걸 수 없었다. "이 프로그램을 처음 본 게 언제냐" 류의 질문이 raw 전 기간 조회에 의존했기 때문이다.

크기 근거는 이렇다. distinct SUID 21,365개 × SUID당 raw 평균 85.5 doc — 신원·최초 정보를 위해 85배의 시계열을 영구 보존하고 있었다. registry는 수십 MB로 같은 질문에 답한다. 부수 효과로 first-seen이 raw 초기화 사고에 면역이 된다. 실제로 7/12 초기화로 현행 first-seen은 이미 한 번 왜곡된 상태였다.

## 변경 범위

| 저장소 | 내용 |
|--------|------|
| 루트(servers/migrations) | `v0039` — `sprocess_registry` 컬렉션 + `{SUID:1}` unique 인덱스, 레거시 `meta.Command` 멱등 회수 |
| servers/service | 라이브 유지 훅(`SprocessRegistryUpdater`) + 백필 스크립트 + prediction ProcName 이관 |
| servers/rest-api | 소비자 3곳 전환(first-seen / aggregated·list first_hour / meta) |

## 핵심 설계 판단 네 가지

### ① 정본을 registry로 삼지 않고 "registry·raw 중 더 이른 쪽"을 쓴다

카드가 raw보다 젊은 국면이 실재한다. 훅이 백필보다 먼저 도는 배포 창이 그렇고, migration 미적용 정책인 개발 서버가 그렇다(백필은 영영 안 도는데 훅은 돌아 "미구축이면 raw" 가정이 성립하지 않는다). raw TTL 이후에는 반대가 되므로, 어느 국면에서도 이른 쪽이 정답이다. 이 설계 덕에 어떤 부분 배포 조합에서도 답이 틀리지 않는다.

### ② merge-min 카드 교체

지각 도착이 흔하다(최대 317h 실측). 그래서 "처음 도착한 doc ≠ 최소 hour"다. 카드 부재 / 더 오래된 hour / 동률 hour에서 백필이 라이브를 덮음, 이 셋 중 하나면 카드를 통째로 교체한다. 세 번째가 없으면 실행 순서마다 source·first_seen이 달라진다. 백필끼리·라이브끼리 동률은 no-op이라 재실행 멱등성이 보존된다.

### ③ Command는 카드에 담지 않는다

Command는 신원이 아니라 그때의 호출이다. 같은 SUID 안에서도 값이 갈리고(distinct 최대 6), `C:/Users/<계정명>/…` 같은 사용자 경로를 품는다(실측 24,186 카드 중 32건). registry는 노드 삭제 cascade 대상도 TTL 대상도 아니라 노드를 지운 뒤에도 영구히 남는다. 사용자 식별 정보를 영구 보존하는 자리에 둘 수 없다. FAMILY도 크기(문서당 약 889B) 때문에 제외했다.

### ④ 백필을 migration이 아니라 스크립트로 둔다

migration에 두면 raw 전량 스캔이 모든 서비스를 정지한 상태의 설치를 몇 시간 막고, 실패 시 설치가 중단되어 서비스가 내려간 채 남는다. 그 사이 agent의 QoS 0 트래픽이 사라진다. 그런데 ①번 설계 덕에 카드가 하나도 없어도 모든 응답이 정답이라, 이번 릴리즈 정확성에는 백필이 불필요하다. 부수 효과로 백필이 service 모듈의 pipeline·필드 목록·meta 생성 함수를 직접 import하게 되어 두 저장소 간 코드 복제가 사라졌다.

## 배포 절차 변경 — 백필은 수동이다

예전 초안은 migration이 백필까지 했다. 이제 분리됐으므로 배포 후 환경마다 한 번 백필 스크립트를 실행해야 한다. 서비스를 켠 채로 아무 때나 돌려도 안전하고 멱등이다(merge-min이 순서 무관). 카드 구성 규칙이 바뀌었을 때만 재구축 옵션을 쓴다.

안 돌려도 지금은 문제가 없지만(카드가 비어도 모든 응답이 정답), 그 상태로 두면 후속 TTL이 raw를 지우는 순간 "진짜 최초"가 사라진다. TTL migration이 백필 완료를 전제조건으로 검사해야 하는 이유다.

## 검증

- 테스트: migrations 14 · service 229(+3 skip) · rest-api 1,520 통과, ruff clean
- 독립 적대 검증 4회. 잡힌 것 중 무거운 것 — 모델 등록 import 누락으로 앱 기동 불가(ruff 자동 저장이 import를 지웠다), 백필이 설치를 몇 시간 막는 구조, 백필/훅의 카드 생성 갈라짐(공백-only meta가 `/meta` 정체성 가드를 우회해 빈 200을 냄), 캐시 상한이 예상 운영점 아래였고 초과 시 전량 clear
- 실측: 백필 2.29M docs → 24,484 SUID를 35초에 처리, 재실행 멱등, 동률 tie-break 5종(라이브→백필과 백필→라이브가 같은 결과), 실 DB HTTP e2e
- 배포 후 확인: `v0039` 수동 적용(러너 미사용, 정책 준수), 백필 실행, 불변식 `raw distinct 24,484 == registry 24,484`, Command·공백-only meta·first_hour 누락 전부 0건, 라이브 훅 로그 정상(seed 24,463 → 신규 SUID upsert, 오류 0·축출 0)

## 남긴 숙제

이 작업은 후속 raw TTL의 전제를 만드는 것까지다. TTL 착수 전에 정해야 할 것이 남는다.

- 백필 완료를 TTL migration의 전제조건으로 검사한다. 순서가 뒤집히면 first-seen이 영구 손실된다.
- registry 파기 정책. HostUrl/ReferrerUrl은 Command와 같은 논리(사용자 식별 정보 + 파기 불가)가 적용되는데 남겼다. first_node는 삭제된 노드 id를 영구 보존하고, 고아 카드 정리 스크립트가 없다.
- proc_name_count 유지 방안. 백필만 채우므로 그 이후 생긴 SUID는 영구히 "미상"이고, raw가 잘린 뒤에는 재계산도 불가능하다.
- 대체 불변식과 백업. `raw distinct == registry`는 TTL 적용 순간 설계상 깨진다. 그 시점부터 registry가 first-seen의 유일본이므로 백업 대상에 넣어야 한다.

*Orange Platform 서버의 프로세스 텔레메트리 차원 테이블 분리에 관한 기술 리포트입니다.*
