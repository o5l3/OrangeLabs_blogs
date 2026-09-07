---
title: "인덱스를 타는데도 매 시작 1.2초 — 부분 인덱스로 죽은 백필 쿼리 잡기"
excerpt: "SQLite에서 인덱스를 타는데도 시작마다 1,179ms를 쓰던 UID 백필 쿼리를 분석했다. 전체 행의 65%가 CompanyUID=0이고 전부 회사명이 빈 파일이라, 인덱스 탐색이 그 3,354행을 전부 짚고 행마다 테이블을 읽어 콜드 캐시에서 HDD 랜덤 읽기 3,354회가 됐다. 쿼리와 C++ 코드는 그대로 두고 인덱스 선언만 부분 인덱스로 바꿔 탐색이 즉시 끝나게 한 과정."
category: tech
date: 2026-08-31
author: kim-tigerj
tags: [SQLite, 부분인덱스, 쿼리최적화, EXPLAINQUERYPLAN, Windows에이전트, 트러블슈팅, Orange Platform]
---

## 현상

에이전트(orange.exe) 시작 시 `CFileList::Create`가 오래 걸린다. 개발 PC 실측(2026-08-31, summary.cdb, 5400rpm HDD):

```
1179.3 ms  CFileList::Create
```

실체는 `CFileList::Patch()`가 시작마다 실행하는 UID 백필 쿼리 `FILELIST_SELECT_COMPANYUID_IS_EMPTY`다. 매칭 0건인데 1,179ms를 썼다.

```sql
SELECT ... from filelist WHERE CompanyName <> '' and CompanyUID = 0 order by EventTime desc
```

## 원인

인덱스를 타는데도 느리다. `EXPLAIN QUERY PLAN`은 `SEARCH filelist USING INDEX filelist.idx.CompanyUID (CompanyUID=?)`.

- 이 PC의 filelist 5,136행 중 3,354행(65%)이 `CompanyUID=0`이고, 전부 회사명이 빈 파일이다. 회사명이 없으면 UID를 계산할 수 없어 이 행들은 영원히 0으로 남는다.
- 인덱스 탐색이 그 3,354행을 전부 짚고, 행마다 테이블을 읽어 `CompanyName <> ''` 검사 후 전부 탈락시킨다. 시작 직후라 캐시가 비어 있어 HDD 랜덤 읽기 3,354회가 됐다.
- 백필할 행이 생기지 않는 한 이 비용이 매 시작 반복된다.

## 조치

`db/summary.json`의 인덱스 선언만 바꾼다. 쿼리 문장과 C++ 코드는 그대로다.

**부분 인덱스 신설**

```sql
CREATE INDEX IF NOT EXISTS [filelist.idx.CompanyUID_patch]
  ON [filelist]([CompanyUID]) WHERE CompanyName <> ''
```

회사명 있는 행(1,782개)만 담고, 그 안에 `CompanyUID=0`이 없으므로 탐색이 즉시 끝난다. 실 DB 사본에서 플랜이 이 인덱스로 전환되는 것을 확인했다.

- 기존 전체 인덱스 `filelist.idx.CompanyUID` 제거. WHERE 절에서 이 인덱스를 쓰는 쿼리는 백필 쿼리 하나뿐이다(다른 등장은 UPDATE의 SET 절).
- 선언 위치를 pre 배열에서 post 배열로 옮긴다. pre는 테이블 스키마 적용 전에 실행되어 완전 신규 DB에서는 filelist가 없어 실패한다. 순서는 CREATE 새 인덱스 → DROP 옛 인덱스(반대로 하면 CREATE 실패 시 인덱스가 하나도 안 남는다).

## 검토 후 제외한 대안

- **복합 인덱스 (CompanyUID, CompanyName)**: 테이블 접근은 없어지나 3,354개 인덱스 항목 스캔은 남는다.
- **회사명 없는 행의 CompanyUID에 별금값(-1) 기록**: UID가 unsigned로 다뤄져 JSON·서버 계약에서 큰 값으로 바뀔 위험이 있다.
- **재시작 시 생략**: 재부팅 시 비용이 그대로 남는다.

## 검증

빌드 시 db.json 재생성 확인, 신규 DB에서 인덱스 존재 확인, 콜드 시작 재측정, 회사명 있는 미계산 행의 백필 동작 확인. 서버가 `get.SQL` 명령으로 내리는 동적 SQL이 기존 전체 인덱스에 의존했는지는 정적 검색으로 배제할 수 없어 명령 이력을 함께 확인했다.

교훈은 단순하다. "인덱스를 탄다"가 "빠르다"를 보장하지 않는다. 인덱스가 짚는 행 수와 그 뒤의 테이블 접근이 진짜 비용이고, 조건에 부합하는 소수만 담는 부분 인덱스가 탐색 자체를 끝낸다.

*Orange Platform 엔드포인트 에이전트의 시작 쿼리 최적화 리포트입니다.*
