---
title: "route 명령어 대신 GUI로: Windows 라우팅 테이블 도구 DHRouteView"
date: 2026-11-17T18:00:00+09:00
draft: true
tags: [windows, rust, networking, open-source]
categories: [remote-management]
summary: "현장 작업자가 라우터를 PC에 꽂으면 라우팅 테이블이 바뀌어 원격 지원이 끊긴다. 명령어를 몰라도 화면에서 라우팅 테이블을 확인하고 바로잡을 수 있도록 Rust로 만든 작은 Windows 도구."
---

## 왜 만들었나

업무 특성상 현장 PC에 원격으로 접속해 지원하는 일이 많다.
그런데 현장 작업자가 사내에서 개발하는 라우터 제품을 PC에 꽂아 두면 라우팅 테이블이 바뀌면서
원격 지원 연결이 불가능한 상태가 되어 버리는 일이 생긴다.

원격으로 들어갈 수 없으니 결국 현장에서 라우팅 테이블을 바로잡아야 하는데,
작업자는 당연히 `route print`, `route delete` 같은 Windows 명령어를 쓸 줄 모른다.
예전에는 Skype, 지금은 Teams로 통화하면서 명령어를 한 글자씩 불러 주는 것도 한계가 있었다.

그래서 이런 상황을 막기 위해, 명령어를 몰라도 라우팅 테이블을 표로 확인하고
화면에서 추가·편집·삭제할 수 있는 작은 GUI 도구를 만들었다.

![DHRouteView 메인 화면. 일반 권한으로 실행해 읽기 전용 모드이고, IPv4 라우팅 테이블이 보인다](/images/dhrouteview/main.png)

## 주요 기능

- **IPv4 / IPv6 라우팅 테이블 조회**: 열 정렬, 필터 지원
- **라우트 추가 / 편집 / 삭제**
  - 게이트웨이를 입력하면 같은 서브넷의 인터페이스를 자동으로 선택
  - 대상 주소에 호스트 비트가 섞여 있으면 네트워크 주소로 정리하고, 적용될 내용을 미리 보여 줌
- **다중 선택과 일괄 삭제**: 탐색기처럼 Ctrl+클릭, Shift+클릭, Ctrl+A, Esc, Delete 키 지원
- **영구 라우트**: `route -p`와 같은 레지스트리 저장소를 쓰도록 구현 (IPv4 + 게이트웨이가 있는 라우트).
  재부팅 후 적용까지는 아직 직접 확인하지 못했다.
- **명령으로 복사**: 우클릭 메뉴에서 선택한 라우트를 같은 라우트를 추가하는 명령으로 복사
  (IPv4는 `route add`, IPv6는 `netsh`)
- 연결이 끊긴 인터페이스의 라우트는 `route print`처럼 기본으로 숨기고, 체크박스로 볼 수 있음

![라우트 추가 대화 상자. 198.51.100.7/24를 넣으면 "적용 내용"에 198.51.100.0/24로 정리된 결과와 자동 선택된 인터페이스가 보인다](/images/dhrouteview/add-dialog.png)

![라우트 3개를 선택한 뒤의 일괄 삭제 확인 대화 상자](/images/dhrouteview/bulk-delete.png)

## 기술적으로 신경 쓴 부분

### 1. route.exe를 호출하지 않는다

`route.exe`를 실행하고 출력을 파싱하는 방식은 간단하지만, Windows 언어 설정에 따라 출력 문구가 달라지고
실패 원인도 정확히 알기 어렵다. 그래서 Windows IP Helper API를 직접 호출했다.

| 동작 | API |
| --- | --- |
| 조회 | `GetIpForwardTable2` |
| 추가 | `CreateIpForwardEntry2` |
| 메트릭 변경 | `SetIpForwardEntry2` |
| 삭제 | `DeleteIpForwardEntry2` |

로캘과 상관없이 동작하고, 실패하면 Win32 오류 코드를 그대로 받아 원인을 번호와 함께 보여 줄 수 있다.

### 2. 메트릭 값이 route print와 다르게 보이는 문제

API가 돌려주는 `Metric`은 **라우트 메트릭**이고, `route print`에 보이는 값은
**라우트 메트릭 + 인터페이스 메트릭**이다. 그대로 표시하면 `route print`와 숫자가 달라 혼란스럽기 때문에,
표에는 합산 값을 보여 주고 추가/편집 폼에서는 라우트 메트릭을 입력받도록 했다.

### 3. 키가 바뀌는 편집은 "추가 후 삭제"

대상(넷마스크 포함)·게이트웨이·인터페이스가 그대로이고 메트릭만 바뀌면 `SetIpForwardEntry2`로 변경한다.
이 중 하나라도 바뀌면 **새 라우트를 먼저 추가하고, 성공했을 때만 기존 라우트를 삭제**한다.
중간에 실패해도 기존 라우트가 사라지지 않게 하기 위해서다.

### 4. 관리자 권한은 필요할 때만

조회는 일반 권한으로 충분하므로 실행할 때마다 UAC 창이 뜨지 않게 했다.
라우트를 변경하려면 앱 안의 "관리자 권한으로 다시 시작" 버튼으로 UAC를 거쳐 다시 실행한다.

### 5. "On-link" 표기

처음에는 게이트웨이가 없는 라우트를 "연결됨 (on-link)"이라고 표시했는데,
연결이 끊긴 인터페이스의 행에도 "연결됨"이라고 나와서 헷갈렸다.
on-link는 링크 상태와 관계없는 개념이라 "On-link"로만 표기하고,
연결이 끊긴 인터페이스의 행은 흐리게 표시하고 인터페이스 이름 옆에 "(연결 끊김)"을 붙였다.

## 개발 환경

- 언어: Rust (edition 2024, MSVC 툴체인)
- GUI: eframe / egui 0.37 + egui_extras (테이블)
- Windows API: `windows` 0.62 크레이트, 레지스트리는 `windows-registry`
- 소스 규모: 약 2,100줄 (`src/*.rs`), 테스트 16개 (순수 로직 14개 + 실제 라우팅 테이블·레지스트리를 읽기만 하는 스모크 테스트 2개)
- 라이선스: MIT

```powershell
cargo build --release    # target\release\dhrouteview.exe
cargo test
```

## 다운로드

- 저장소: https://github.com/pdh0085/DHRouteView
- 릴리스 v0.1.0: https://github.com/pdh0085/DHRouteView/releases/tag/v0.1.0
  - `DHRouteView.exe`: 설치 과정 없이 실행하는 exe 파일 하나
  - `DHRouteView-v0.1.0-windows-x64.zip`: 실행 파일 + LICENSE + README

코드 서명이 없어 처음 실행할 때 Windows SmartScreen 경고가 나올 수 있다.
라우트를 변경하려면 관리자 권한이 필요하다.

## 마무리

개발자에게는 명령어 몇 줄이면 되는 일이지만, 명령어를 모르는 현장 작업자에게는 쉽지 않은 일이다.
화면에서 라우트를 보고 클릭 몇 번으로 정리할 수 있으니, 원격 지원이 끊기는 상황에 훨씬 쉽게 대응할 수 있을 것으로 기대한다.
앞으로는 라우트 목록 내보내기/가져오기 같은 기능을 추가해 볼 생각이다.
