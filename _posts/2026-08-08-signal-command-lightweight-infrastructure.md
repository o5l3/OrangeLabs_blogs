---
title: "경량 명령(Signal Command) 인프라화 — 결과 집계 없는 표준 명령 통로와 '경량 ≠ 무책임' 원칙"
excerpt: "결과 집계가 필요 없는 경량 명령을 단일 채널(command:signal + dispatch_signal)로 통합하고 그 위에 노드 원격 제어(재시작/업데이트/재부팅/종료)를 단건·그룹으로 개통. 명령의 '무게'를 결과수집·배달보장·감사 3축으로 재정의한 인프라 구축 기록."
category: tech
date: 2026-08-08
author: smahn9123
tags: [명령체계, Redis, MQTT, RBAC, 감사로그, 백엔드, Orange Platform]
---

## 개요

결과 집계가 필요 없는 경량 명령을 **표준 인프라(`command:signal` 단일 채널 + `dispatch_signal`)로 통합**하고, 그 위에 노드 원격 제어(Agent 재시작/업데이트, PC 재부팅/종료)를 단건·그룹으로 개통했다. 하드코딩되어 있던 경량 경로(reauth / rule_reload)를 단일 배관으로 흡수해, 새 경량 액션 추가 시 채널·핸들러를 복제하지 않는다. (구현 완료 · 배포 · 라이브 검증됨.)

## 명령의 '무게' — 3축 재정의 (as-built)

| 축 | 추적 명령(T2) | 경량 신호/액션(T0/T1) |
|-----|--------------|----------------------|
| A. 결과 수집·집계 | Redis Hash + WS | 없음 (fire-and-forget) |
| B. 배달 보장 | expiration까지 존속·retry | transient (오프라인 노드는 놓침) |
| C. 감사·책임추적 | Command 문서 | T1은 audit_log 1행 필수 |

원칙 **'경량 ≠ 무책임'**을 구현으로 못박음: 결과 집계(A)를 버리는 것과 감사(C)를 버리는 것은 별개다. PC를 끄는 T1 액션은 결과 수집이 없어도 NodeLog 감사를 반드시 남긴다.

| 티어 | 결과수집 | 감사 | 구현 |
|------|----------|------|------|
| T0 신호 | ✗ | 선택 | reauth·rule_reload → `command:signal`로 통합 |
| T1 경량 액션 | ✗ | 필수 | 노드 제어(재시작/업데이트/재부팅/종료) |
| T2 추적 명령 | ✓ | ✓ | 기존 `POST /v3/command` (변경 없음) |

## API 변경 영향도 — 신규 엔드포인트

모두 매니저 인증 + `node_control` 기능 권한 게이팅. 결과 수집이 없는 경량 signal — 응답은 '발송(dispatched)'까지만 보장한다.

| 메서드 | 경로 | Agent verb | 설명 |
|--------|------|-----------|------|
| POST | `/v3/node/{id}/control/restart-agent` | `@restart` | Agent 재시작 |
| POST | `/v3/node/{id}/control/reboot` | `@reboot` | PC 재부팅(즉시 강제) |
| POST | `/v3/node/{id}/control/shutdown` | `@shutdown` | PC 종료(전원 off·원격 재기동 불가) |
| POST | `/v3/node/{id}/control/update` | `@update` | Agent 즉시 업데이트(non-force·버전 다를 때만) |
| POST | `/v3/node/control/bulk/{action}` | — | 그룹 실행 — `nodeIds[]`로 다수 노드 일괄 |

기존 감사 조회(`GET /v3/audit-logs`·`/count`)로 노드 제어 이력을 조회한다. 다른 API 계약 변경 없음.

## 주요 구현

### 서버 (rest-api)

- `CommandService.dispatch_signal(action, command_data, node_ids, ...)` 신설. `trigger_reauth_command`·`trigger_rule_policy_refresh`는 이 배관의 얇은 caller로 강등.
- `NodeControlCaseBase`(권한→노드조회→dispatch→감사 공통 흐름) + 액션별 서브클래스 4종. 새 액션 = 서브클래스 + 엔드포인트만 추가.
- **그룹(벌크)**: 그룹코드가 아니라 노드 ID 목록을 받는다(소속 이동 후 재인증 전이면 그룹 MQTT 토픽이 낡아 오배송·감사 불가). 서버는 프런트 목록을 맹신하지 않고 중복제거→존재/제어가능 노드만 추려 발송. 감사는 요약 1행(group_label·target_count·node_ids).
- **감사 정직성**: 신호 유실(receiver=0) 시 `result=false`. 감사 목록 응답은 비대 필드(node_ids)를 50건으로 잘라 응답 폭주 방지(원본 보존).

### 외부 서비스 (service)

- 용도별 채널(`command:reauth`·`rule:policy_changed`)을 `command:signal` 단일 채널 + `_handle_signal_message` 통합 핸들러로 흡수. 레거시 채널은 이후 완전 제거(발행자가 없어 죽은 구독이었음).
- 신호 로그에 회선 형태(command/verb) 기록. **MQTT publish 반환값(rc)까지 확인** — 브로커 미연결/발행 실패를 '발송 완료'로 숨기지 않고 ERROR로 정직 기록(그룹으로 폭발 반경이 커져 추가).

### 매니저 웹 (manager-web)

- '전체 노드' 위젯 우클릭: 단건(노드 행) + 그룹(좌측 트리 그룹의 online 노드 일괄).
- 노드 상세 위젯 '액션' 드롭다운: 같은 액션을 한 노드에 집중한 화면에서 바로 실행.
- 공용 훅 `src/hooks/useNodeControl.tsx`: 권한 판정 + 단건/그룹 mutation + 토스트를 폼 상태 비의존으로 추출. 우클릭·상세 드롭다운 공유.

## 거버넌스 (T1 필수 조건 — 구현됨)

| 항목 | 구현 |
|------|------|
| RBAC | `node_control` FeaturePermission. 파괴적이라 admin-only 기본 시딩(fail-closed, super_admin/root 우회). viewer는 제어 불가. |
| 감사 | NodeLog(category=node) 1행. 유실도 `result=false`로 정직 기록. |
| 서명 | `dispatch_signal`이 `sign_data(command_data)` 상속. 실측으로 서명 유효 확인. |
| 확인·안전 | UI 확인 다이얼로그. 그룹 실행 spread(재시작 600s/전원 60s/업데이트 1800s) payload 동봉. |

## 배포·검증

- **배포 순서**: Redis Pub/Sub 미존속 → service를 rest-api보다 먼저.
- **스테이징 라이브 검증**: 단건 재시작·재부팅 실동작, 그룹 재시작 실동작(발송 4초 뒤 `Agent.start` 재시작 확증). 감사 DB 실측으로 5대 항목 적재 확인. MQTT 캡처 payload 전 필드 + 서명 암호 검증.
- **Agent 업데이트(`@update`)**도 동작 확인됨. 초기엔 Agent의 `@update` 분기가 정확일치 `"@update\n"`만 받아 개행 없는 서버 payload가 안 먹었으나, 후속 수정으로 `@update`/`@update\n` 둘 다 받도록 하고 노드 다수에 배포됨. `@update2`(force)는 접두 매칭이라 `@update` 분기보다 앞에 두어 충돌 방지.

## 주의사항 / 후속

- **spread 미해석**: 그룹 실행 시 spread를 payload에 실으나 현재 Agent가 해석하지 않아 실제 동시 실행 완화는 아직 없다(향후 Agent 지원 시 자동 적용).
- **qos=0 한계**: publish rc 확인으로 '브로커에 못 넣은' 것은 잡지만, rc=0이 agent 수신까지 보장하지는 않는다(fire-and-forget 설계 전제).
- **org 스코프 미검사**(기존 이슈, 벌크가 확대): 부서 스코프로 준 역할도 전사 제어 가능. 별도 이슈.

## 변경 범위 (3개 서브모듈)

| 위치 | 변경 |
|------|------|
| `app/services/command_service.py` | `dispatch_signal` 신설, `trigger_*` 재작성 |
| `servers/service/command.py` | `command:signal` 통합 핸들러 + 레거시 채널 제거 + publish rc 정직 확인 |
| `app/api/v3/node.py` | 노드 제어 8개 엔드포인트(단건 4 + 벌크 4) |
| `app/usecases/node/control/` | `NodeControlCaseBase` + 액션별 UseCase 4종 |
| `app/models/mongo/audit_log/node_log.py` | NodeLog(category=node) + Action/Data |
| `app/models/mongo/store/feature_permission_store.py` | `node_control` admin-only 시딩 |
| `web/manager-web/src/hooks/useNodeControl.tsx` | 공용 훅(신규) |

---

*Orange Platform 명령 인프라 표준화 기술 리포트입니다.*
