---
title: "VM 노드가 재기동할 때마다 자기 자신과 문서가 분열되는 이유 — 종료 POST의 티켓 회전 억제"
excerpt: "하드웨어 앵커가 없는 VM 노드가 정상 종료 후 재기동할 때 새 문서로 분열(중복 노드)되던 결함을 추적한 기록. 에이전트가 online일 때만 새 ticket/guid를 저장하는데 종료(offline) POST에서 서버가 회전시켜 desync가 남던 원인을 규명하고, 서버가 online일 때만 회전하도록 게이팅해 원천 차단했다."
category: tech
date: 2026-07-21
author: smahn9123
tags: [노드 인증, VM, 분산 신원, ticket/guid 회전, desync, FastAPI, MongoDB, Orange Platform]
---

## 개요

VM 노드가 정상 종료 후 재기동할 때 자기 자신과 문서가 분열되던 결함을 수정했다. **서버가 `status=online`일 때만 ticket/guid를 회전하도록** 바꿔 desync를 원천 차단한다.

## 근본 원인

에이전트는 online(`bOnline`)일 때만 인증 응답의 새 ticket/guid를 저장한다(`ApplyServerProfile`). 종료(offline) POST에서 서버가 회전하면 에이전트는 그 값을 버리고 꺼지므로 **서버 ticket만 앞서가 desync가 남고**, 다음 기동 때 같은 VM이 stale ticket을 제시 → 매칭 실패 → 새 문서로 분열된다.

하드웨어 앵커가 없는 VM에서만 표면화된다(ticketb 폴백 제거 · guid 매회전 조합의 부작용).

## 수정 내용

`app/usecases/node/node_auth_case.py` — `_rotation_targets()` 추가. 회전값 확정 직후 status로 게이팅한다.

```python
if status != NodeStatus.ONLINE:
    next_ticket, guid_new = ticket, guid   # 회전 억제 (정체성 보존)
```

종료 POST는 무회전 no-op으로 현재 ticket에 매칭해 status만 offline으로 갱신 → 서버 ticket 유지 → 다음 기동 정상 매칭. 클론 분리는 online 부팅에서 일어나므로 영향이 없다.

## 검증

- node 인증 테스트 128개 전부 통과 (신규 4개: online→회전 / offline·uninstall·빈값→억제). ruff check·format 클린.
- 전 경로 논리 검토: VM/비-VM × POST/PUT × 신규/기존 × online/offline. 클론 분리·PUT·비-VM 경로 모두 불변 확인.
- 잔여 리스크(인지): online POST 응답이 네트워크에서 유실되면 드물게 fork 가능 — 현행도 fork되던 것이라 회귀는 아니다. ticketb 매칭 부활은 클론 분리를 깨므로 불가.

## 배포 / 정리

- 커밋 후 rest-api 컨테이너 재기동, 컨테이너 내부 코드에 fix 반영 확인 → 라이브.
- 유령 문서 정리: 운영 삭제 경로(`DELETE /v3/node`)로 중복 노드 문서를 아카이브 처리.
- 동시 실행 클론 풀·물리 앵커가 상이한 노드 등 일부는 배포로 재-fork가 멈춘 뒤 동결 유령만 추가 정리하기로 하고 의도적으로 제외.

관찰 신호: 이후 offline POST가 온 노드의 감사 배열에 `rotation suppressed …` 라인이 찍히면 실전 동작 확인.

## 설계 계보

이번 수정은 VM 클론을 강제로 분리하는 기존 장치(설계 계보상의 여러 선행 이슈)를 훼손하지 않고 **종료 시 self-fork만 제거**한 것이다. 후속(선택)으로 "에이전트 종료 시 재인증 POST 생략"을 검토할 수 있으나, 서버 가드가 있으면 필수는 아니다.

---

*이 글은 Orange Platform 노드 인증·분산 신원 관리의 분석 및 트러블슈팅 리포트입니다.*
