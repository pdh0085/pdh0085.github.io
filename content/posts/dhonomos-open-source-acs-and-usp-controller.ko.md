---
title: "Dhonomos: TR-069 ACS와 USP Controller를 Rust 실행 파일 하나로"
date: 2026-11-06T19:00:00+09:00
draft: false
tags: [tr-069, tr-369, rust, open-source]
categories: [remote-management]
summary: "TR-069(CWMP) ACS와 TR-369(USP) Controller를 Rust 프로그램 하나로 만든 Dhonomos를 Apache-2.0으로 공개했다. 왜 만들었는지, 지금 무엇을 하는지, 아직 무엇을 못 하는지."
---

가정용 공유기나 LTE/5G CPE, ISP 운영 쪽 일을 한다면 원격 장비 관리의 두 표준을 알 것이다.
현장 CPE 대부분이 쓰는 SOAP-over-HTTP 프로토콜 **TR-069**(CWMP), 그리고 그 후속으로 MQTT,
WebSocket, STOMP 위에서 도는 **TR-369**(USP)다. TR-069의 서버 쪽을 ACS, USP의 서버 쪽을
Controller라고 부른다.

ACS와 USP Controller를 Rust 프로그램 하나로 만든 **Dhonomos**를 Apache-2.0 라이선스로
공개했다.

**<https://github.com/pdh0085/dhonomos>**

![Dhonomos 웹 UI의 장비 목록. 화면의 장비는 시뮬레이터다](/images/dhonomos/devices.png)

왜 만들었는지, 지금 무엇을 하는지, 아직 무엇을 못 하는지 적는다.

## 왜 또 ACS인가

나는 대만의 공유기·LTE/5G CPE 제조사 Lightspeed International에서 소프트웨어를 개발한다.
펌웨어를 시험하려면 사업자 서버가 하는 방식 그대로 장비와 대화해야 한다. 파라미터를 읽고,
설정을 바꾸고, 펌웨어를 내려보내고, NAT 뒤에 있는 장비를 깨운다. 그래서 랩에서 돌릴 수 있고,
코드를 읽을 수 있고, 장비가 이상하게 굴 때 고칠 수 있는 서버가 필요했다.

오픈소스 선택지는 적었다. 상용 ACS는 비싸고 닫혀 있다. 가장 잘 알려진 오픈소스 ACS는
AGPL이라 많은 회사가 운영하거나 고치기를 꺼린다. 그리고 USP Agent(오픈소스 obuspa 같은)는
이미 제품에 들어가 있는데, 작은 팀이 실제로 배포할 수 있는 오픈소스 USP Controller는 드물다.

그래서 Dhonomos는 우리 랩의 시험 보조 도구로 시작해 완전한 서버로 자랐다. 회사가 오픈소스
공개에 동의했다.

## 하는 일

**CWMP ACS**

- TLS 위의 TR-069 Amendment 6 세션(CWMP 1.0-1.4), Digest 또는 Basic 인증.
- 장비에 묶인 장비별 자격증명. 첫 접속 때 발급할 수도 있다. 발급한 뒤로는 그 장비에 공용
  자격증명이 더는 통하지 않는다.
- Zero-touch 등록: 새 장비는 첫 Inform 때 정책(전부 받기, 격리, 사전 등록만, OUI 허용 목록)에
  따라 전체를 읽는다.
- Connection Request를 HTTP로, 내장 STUN 서버를 쓰는 UDP로, 내장 XMPP 서버를 쓰는 **XMPP**로
  보낸다. XMPP는 통신사 NAT(CGNAT) 뒤의 CPE에 중요하다. 장비가 서버로 TCP 연결을 열어 두고,
  깨우는 신호가 그 연결로 내려가며 전달 확인도 받는다.
- 펌웨어·설정 파일용 내장 파일 서버. 서명된 링크와 대역폭 제한이 있다.

**USP Controller**

- MTP: MQTT 5(Mosquitto 같은 외부 브로커 또는 Dhonomos 내장 브로커), 양방향 WebSocket,
  내장 브로커를 쓰는 STOMP 1.2.
- Agent 인증서용 자체 CA. 발급, 만료 목록, 폐기를 지원한다. 인증서가 있는 EndpointID는
  더 약한 MTP로 가로챌 수 없다.

**운영**

- 재시도가 있는 작업 큐, 프리셋(YAML 또는 웹 UI), 샌드박스 안에서 도는
  [Rune](https://rune-rs.github.io/) 프로비저닝 스크립트. Rune은 Rust에 넣어 쓰는 작은 언어다.
- 카나리 단계와, 실패 장비가 많으면 자동으로 멈추는 기능이 있는 대량 작업·펌웨어 캠페인.
- CWMP와 USP가 같은 결과 형식을 쓰는 진단: ping, traceroute, DNS, TR-143 속도 시험,
  주변 Wi-Fi.
- 설정 백업, 변경 감지, 복원, 장비 교체.
- 서버에서 관리하는 장비 WireGuard 터널, 선택 사항인 허브 프로그램.
- OpenAPI 3.1 문서가 있는 REST API, 실행 파일에 내장된 웹 UI(SvelteKit), 역할, 팀별 장비
  범위, OpenID Connect SSO, TOTP 2단계 인증, 감사 로그.

서버 전체가 실행 파일 하나와 PostgreSQL이다. 컨테이너 이미지는 빈 베이스 이미지 위의 정적
musl 바이너리 하나로, 약 22 MB다.

## 성능

12코어 Xeon E5-2630(2.3 GHz)에 PostgreSQL을 같은 호스트에 두고, 시뮬레이터로 **가상 CPE
5만 대가 초당 약 2,000건의 정기 Inform**을 60초 동안 보냈다. 오류는 없었고, Inform p50
9.9 ms, p99 441 ms, 서버 메모리는 **113 MB**였다. 이 측정은 DB 풀과 동시 세션 한도를 기본값보다
늘린 설정(연결 96개, 동시 세션 4,096)이다. 기본값으로는 같은 부하에서 세션의 5.7%가 503으로
거절된다. 시뮬레이터와 벤치마크 도구가 저장소에 있으니(`crates/simulator`,
`cargo xtask bench`) 같은 측정을 직접 해 볼 수 있다.

## 왜 Rust인가

관리 서버는 ISP의 인터넷 경계에 놓이고, 통제할 수 없는 수만 대 장비의 입력을 파싱한다.
거기서는 메모리 안전성이 중요하고, 작고 예측 가능한 자원 사용량도 중요하다. Rust는 둘 다 줬고,
필요한 부분마다 좋은 생태계가 있었다. 네트워크 쪽은 tokio와 axum, PostgreSQL은 컴파일 때
쿼리를 검사하는 sqlx, USP protobuf는 prost, TLS는 rustls다. `unsafe`는 한 crate만 빼고 전부
금지했고, 그 하나에도 문서화한 `getsockopt` 호출 하나뿐이다. 프로토콜 파서에는 퍼징 대상이 있다.

## 아직 못 하는 것

솔직하게 먼저 적는다.

- **1.0 전**이다. 아직 릴리스가 없어 소스에서 빌드해야 한다. 플래그, API, DB 스키마가 아직
  바뀔 수 있다.
- **외부 보안 리뷰**는 아직 받지 않았다. 내부 리뷰만 했다.
- 운영 가이드는 한국어이고, README와 설치 가이드는 영어다.
- 완전한 멀티 테넌시는 아직 없다.
- OpenWrt 기반 CPE와 시뮬레이터로 시험했다. 다른 제조사 장비는 우리가 아직 못 본 동작을
  할 수 있다. 제보를 환영한다.

## 써 보기

```bash
git clone https://github.com/pdh0085/dhonomos.git && cd dhonomos
docker build -t dhonomos:dev .
cp deploy/dhonomos.env.example deploy/dhonomos.env   # fill in the secrets, CPE credentials, ACS cert
docker compose -f deploy/docker-compose.yml up -d
```

그다음 CPE의 ACS URL을 이 서버로 맞추면 된다.
[설치 가이드(영어)](https://github.com/pdh0085/dhonomos/blob/main/docs/guide/install.en.md)와
[한국어 설치 가이드](https://github.com/pdh0085/dhonomos/blob/main/docs/guide/install.md)가
TLS, 첫 CPE, 첫 USP Agent까지 안내한다.

## 이름

Dhonomos는 "DH"(내 이니셜)에 그리스어 *oikonomos*를 붙인 이름이다. oikonomos는 집안 살림을
맡은 관리인이라는 뜻이고, *economy*라는 말의 뿌리다. 가입자 집 안의 장비를 돌보는 서버에
어울리는 일이라고 생각했다.

지금 ACS를 운영하거나, CPE를 만들거나, USP Controller를 찾고 있다면 무엇이 필요한지 듣고 싶다.
GitHub 이슈로 알려 주면 된다.
