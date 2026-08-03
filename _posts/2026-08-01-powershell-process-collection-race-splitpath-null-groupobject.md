---
title: "PowerShell 프로세스 수집이 간헐적으로 부분 실패하는 이유 — 조회 후 종료 경합과 Split-Path null"
excerpt: "1:N 프로세스 수집 템플릿이 25회 중 4회 부분 실패하던 원인을 추적한 기록. Get-Process를 Where-Object로 거른 뒤 그 프로세스가 종료되면 .Path가 null이 되고, Split-Path가 null을 받아 그 항목이 깨지던 경합을 규명한다. Group-Object가 그룹 이름($_.Name)에 이미 담고 있는 경로 문자열을 재사용해 프로세스 객체를 다시 읽지 않도록 고쳐 부분 실패를 0으로 만들었다."
category: tech
date: 2026-08-01
author: kim-tigerj
tags: [PowerShell, Get-Process, 경합 조건, Split-Path, Group-Object, 프로세스 수집, 트러블슈팅, Orange Platform]
---

## 개요

1:N 프로세스 수집 템플릿 두 건이 간헐적으로 부분 실패했다. **조회 뒤 그 프로세스가 종료되면 경로가 비어 `Split-Path`가 터진다.** 실측 4/25회.

## 증상

에이전트가 `AGENT_ERROR_PS_PARTIAL(-20)`으로 응답한다. 값은 대부분 오지만 일부 항목이 빠지고 아래 오류가 함께 기록된다.

```text
'Path' 매개 변수가 null이므로 인수를 해당 매개 변수에 바인딩할 수 없습니다.
```

에이전트가 부분 실패를 구분해 알리기 전에는 code 0으로 나가 묻혀 있었다.

## 원인

경로를 프로세스 객체에서 다시 읽는다. `Where-Object`를 통과한 뒤 그 프로세스가 끝나면 `.Path`가 null이 되고, `Split-Path`가 null을 받지 못해 그 항목이 깨진다.

```powershell
Get-Process | Where-Object {$_.Path} | Group-Object -Property Path | ForEach-Object {
  $f = Get-Item $_.Group[0].Path -ErrorAction SilentlyContinue
  [PSCustomObject]@{ ... FileName = Split-Path $_.Group[0].Path -Leaf; ... Path = $_.Group[0].Path }
}
```

## 조치

`Group-Object -Property Path`는 그룹 이름(`$_.Name`)에 경로 문자열을 이미 담고 있다. 프로세스 객체를 다시 읽지 않고 그 값을 쓴다.

```powershell
Get-Process | Where-Object {$_.Path} | Group-Object -Property Path | ForEach-Object {
  $p = $_.Name
  if ($p) {
    $f = Get-Item $p -ErrorAction SilentlyContinue
    [PSCustomObject]@{ ... FileName = Split-Path $p -Leaf; ... Path = $p }
  }
}
```

- `$p = $_.Name` — 경로를 그룹 이름에서 **한 번만** 가져온다.
- `if ($p)` — 빈 값이면 건너뛴다.
- `Split-Path`·`Get-Item`·`Path`를 모두 `$p`로 바꾼다.
- 특정 프로세스 템플릿은 `FileVersionInfo`도 `$_.Name`의 파일에서 읽는다.

## 검증

프로세스가 뜨고 지는 부하를 걸고 25회 교대 실행.

| 템플릿 | 상태 | 부분 실패 | 결과 건수 |
|---|---|---|---|
| 전체 프로세스 | 현재 | 4/25 | 100건 |
| 전체 프로세스 | 수정 후 | 0/25 | 100건 |
| 특정 프로세스 | 현재 | 0/25 | 6건 |
| 특정 프로세스 | 수정 후 | 0/25 | 6건 |

결과 건수와 형태는 그대로이고 부분 실패만 사라진다.

*Orange Platform 프로세스 수집 템플릿 트러블슈팅 리포트입니다.*
