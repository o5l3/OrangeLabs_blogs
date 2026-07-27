---
title: "무인 업데이트 스크립트가 DB 마이그레이션에서 무한 대기하는 이유 — 삼켜진 stdin 프롬프트 추적"
excerpt: "unattended 업데이트 스크립트가 DB migration 단계에서 멈추던 문제를 추적한 기록. must_task가 stdout만 리다이렉트하고 stdin은 운영자 TTY에 붙여둔 탓에, migration의 input() 프롬프트가 화면에 안 보인 채 입력을 기다리며 무한 대기하던 원인을 규명하고, 무인 실행자가 stdin을 /dev/null로 차단하는 계층 분리로 해결했다."
category: tech
date: 2026-07-23
author: smahn9123
tags: [셸 스크립트, TTY, stdin, DB 마이그레이션, 무인 업데이트, 트러블슈팅, MongoDB, Orange Platform]
---

## 개요

무인(unattended) 업데이트 스크립트 `package/install/update.sh`가 DB migration 단계에서 무한 대기에 빠져, 특정 서버에서 마이그레이션이 `v0005` 이후 진행되지 않고 멈추는 문제가 발생했다.

## 증상

- `_migrations` 컬렉션에 `v0005`까지만 기록되고 이후 멈춤.
- 콘솔에는 `DB 마이그레이션 (migrations) .........` 점선만 표시되고 진행 없음.
- 실제 확인: migration 프로세스가 `n_tty_read` → `tty_read` (fd0 → /dev/pts) 스택에서 **TTY 입력 대기 중**.

## 근본 원인

- `update.sh`의 `must_task`는 태스크 stdout/stderr를 임시 로그로 리다이렉트하고 실패 시에만 출력한다 (`"$@" > "$log" 2>&1`).
- migration `v0006_seed_customer_manager._resolve_customer_id()`는 `sys.stdin.isatty()`가 참이면 `input()`으로 고객 계정 ID를 묻는다.
- `must_task`는 stdout만 캡처하고 **stdin은 운영자 TTY로 남겨둔다.** 그 결과 `input()`은 정상적으로 입력을 기다리는데, 프롬프트 텍스트는 로그로 삼켜져 화면에 안 보여 운영자가 인지하지 못한 채 무한 대기에 빠진다.

즉, 무인 스크립트가 stdin을 대화형 TTY에 붙여둔 것이 결함이다.

## 수정

계층 분리 — **무인 실행자가 "대화형 입력 없음"을 명시적으로 선언**하도록 했다.

- `package/install/update.sh` (`must_task`): 명령 실행 시 stdin을 `/dev/null`로 차단 → `"$@" </dev/null > "$log" 2>&1`. `v0006`뿐 아니라 향후 모든 태스크가 stdin 블록에서 보호된다.
- `servers/migrations/versions/v0006_seed_customer_manager.py`: 판정 기준을 `stdin.isatty()`로 유지(원복)하고, `update.sh`와의 규약을 주석으로 명시. stdout 캡처 여부로 판단하지 않는다(출력 스트리밍 개선 시 재발/오작동 방지).

효과:

- `setup.sh install` (stdin=tty): 대화형 프롬프트 정상 유지.
- `update.sh` (stdin=/dev/null): 프롬프트 없이 비대화형 기본값(admin) 사용.
- `CUSTOMER_ID` 환경변수 override는 그대로 최우선.

## 검증 (Docker, 실 MongoDB + 실제 v0006 + 실제 must_task)

`mongo:8.0` 컨테이너에 실제 `v0006.Migration.up()`과 `update.sh`에서 verbatim 추출한 `must_task`를 진짜 pty(TTY stdin) 아래에서 실행했다. 총 23개 체크 전부 PASS.

- 수정 전 wrapper + tty stdin → 무한 대기 재현(프로덕션 증상 일치).
- 수정 후 wrapper → 무한 대기 없이 admin 시드, `role=super_admin`, `must_change_password=True`.
- `setup.sh install` 대화형 경로 → 프롬프트 표시 + 입력 ID 반영(회귀 없음).
- `CUSTOMER_ID` override, 멱등성(2회 실행 시 중복 없음) 정상.

## 변경 파일

- `package/install/update.sh` — `must_task` stdin `/dev/null` 차단
- `servers/migrations/versions/v0006_seed_customer_manager.py` — stdin 기준 유지 + 규약 주석

---

*이 글은 Orange Platform 서버 설치·업데이트 자동화 파이프라인의 트러블슈팅 리포트입니다.*
