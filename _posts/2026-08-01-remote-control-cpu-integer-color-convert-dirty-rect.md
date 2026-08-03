---
title: "가상 GPU 노드 원격제어 CPU 부하 최소화 — BGRA→NV12 색변환 정수화 + dirty-rect 부분 갱신"
excerpt: "하드웨어 인코더가 없는 가상 GPU(VMware SVGA 3D) 노드의 원격제어 시 CPU 부하를 줄인 2단계 최적화 기록. 소프트웨어 BGRA→NV12 색변환을 부동소수·픽셀당 나눗셈에서 정수 고정소수점으로 재작성해 MSVC /O2의 SSE2 자동 벡터화를 유도하고, DXGI dirty-rect로 바뀐 영역만 GPU→CPU 복사·변환하도록 바꿔 실측 CPU를 10 → 6 이하로 단계적으로 낮췄다."
category: tech
date: 2026-08-01
author: kim-tigerj
tags: [원격제어, DXGI, NV12, SSE2, dirty-rect, 성능 최적화, VMware, Orange Platform]
---

## 개요

가상 GPU(VMware SVGA 3D) 노드는 하드웨어 인코더가 없어 소프트웨어 경로(CPU 색변환 BGRA→NV12 + 소프트웨어 H.264)를 탄다. 원격제어 시 대상 노드의 CPU 부하를 최소화하기 위한 2단계 최적화다.

## 작업 1 — CPU 색변환 정수화 (SSE2 자동 벡터화)

1:1(다운스케일 없음) 경로의 **BGRA→NV12 변환을 부동소수·픽셀당 정수 나눗셈에서 정수 고정소수점으로 재작성**했다. MSVC `/O2`가 이 단순 정수 루프를 SSE2로 자동 벡터화한다. 색 계수는 기존 float 경로와 시각적으로 동일하다(중립·순색 검증).

**결과**: 실측 CPU 약 **10 → 6**, 색감 변화 없음.

## 작업 2 — dirty-rect 부분 갱신

DXGI `GetFrameDirtyRects`/`GetFrameMoveRects`로 **바뀐 영역의 바운딩 박스만 GPU→CPU 복사**(`CopySubresourceRegion`)하고 그 박스만 변환한다. 지속 NV12 버퍼 위에 덧쓰므로 읽기·변환 CPU가 화면 전체가 아니라 **변화 크기에 비례**한다.

안전장치:

- move rect가 있거나 메타데이터 없음·초기·재생성·해상도 변경 시 전체 변환 폴백.
- 부분 갱신 120회마다 전체 새로고침(dirty 누락 자가치유).

**결과**: 작업 1 위에서 CPU 추가 감소 확인.

## 검증

- **VM 실측**: 색감 동일, CPU 단계적 감소(10 → 6 → 추가 감소).
- **교차검증**: 신규 dirty-rect 로직에 결함 없음(비회전 1:1에서 타당). even 정렬 ↔ NV12 2x2 그리드 일치, stale 영역 미읽기, 지속 버퍼 초기화, move/dirty 버퍼 계약 모두 확인.

## 남은 이슈 (별도, 기존 결함)

세로(회전 90/270) 모니터에서 CPU 변환 경로가 깨진다. `captureStaging`(width×height)과 `captureTex`(height×width) 크기 불일치 + CPU 변환에 회전 로직 없음. 이번 변경이 만든 것이 아니라 기존부터 있던 결함으로, 부하를 줄인 빌드로 복귀 시 재확인 예정이다.

*Orange Platform 원격제어 성능 최적화 기술 리포트입니다.*
