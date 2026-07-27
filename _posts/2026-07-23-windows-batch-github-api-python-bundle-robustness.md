---
title: "빌드가 GitHub API 불안정에 무한 대기하던 문제 — 배치 스크립트의 Python 번들 단계 견고화"
excerpt: "makepackage.bat의 Python standalone 번들 단계가 GitHub API/네트워크 불안정 시 무한 대기하거나 빈 URL을 curl에 넘겨 빌드를 중단시키던 문제를 수정한 기록. cmd for /f 파싱을 걷어내고 PowerShell 단일 블록으로 통합해 타임아웃·재시도·폴백 URL·크기 검증을 갖췄다."
category: tech
date: 2026-07-23
author: smahn9123
tags: [Windows 배치, PowerShell, GitHub API, 빌드 견고화, 타임아웃, 재시도, Orange Platform]
---

## 개요

`makepackage.bat`의 "Python 3.13 standalone bundle" 단계가 GitHub API/네트워크가 불안정할 때 무한 대기하거나, 빈 버전/URL을 반환해 `curl: option : blank argument`로 빌드가 중단되던 문제를 수정한다.

## 원인

기존 구현은 `for /f "delims=|"`로 인라인 PowerShell 출력(`버전|URL`)을 파싱했고, 버전 추출에 PowerShell `$matches` 자동 변수를 사용했다. GitHub API 응답이 지연/불안정하면:

- `Invoke-RestMethod`에 타임아웃이 없어 무한 hang.
- 버전 파싱이 비어(`$matches` 클로버링 등) `%PY_VER%`가 공백, `%PY_URL%`이 빈 문자열이 되어 curl에 빈 URL을 전달 → 빌드 중단.

## 수정

resolve → download → 검증 전체를 **PowerShell 단일 블록으로 통합**하고, cmd `for /f`/`delims=|` 파싱을 제거했다.

- 버전 추출을 `$matches` → `[regex]::Match(...).Groups[1]`로 결정적 처리
- `Invoke-RestMethod`/`Invoke-WebRequest`에 `-TimeoutSec` 부여 (무한 hang 방지)
- API·다운로드 각각 최대 4회 재시도 + User-Agent 헤더 + TLS1.2 강제
- API 완전 실패 시 검증된 고정 폴백 URL로 자동 전환 (빌드 무중단)
- 다운로드 파일 크기(>5MB) 검증으로 손상/HTML 오류응답 방어

`makepackage.sh`는 `uv python install` 경로라 동일 취약점이 없어 수정이 필요 없다.

## 변경 범위

- `makepackage.bat` (Python 3.13 번들 단계)

---

*이 글은 Orange Platform 패키징 빌드 스크립트의 견고화 트러블슈팅 리포트입니다.*
