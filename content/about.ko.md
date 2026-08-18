---
title: "소개"
hidemeta: true
comments: false
showtoc: false
---

OpenWrt와 Yocto 기반의 셀룰러 CPE(5G/4G 라우터) 펌웨어를 만드는 엔지니어입니다.
이 블로그에는 현장에서 실제로 부딪힌 문제와 그 해결 과정을 기록합니다.

주로 다루는 것들:

- OpenWrt / Yocto 기반 라우터 펌웨어 (Qualcomm IPQ807x 등)
- 셀룰러 모뎀 연동: Quectel, Telit Cinterion, Sierra Wireless, u-blox 등 —
  AT 커맨드, QMI / MBIM, USB 네트워킹 모드 (RNDIS / ECM / NCM)
- TR-069 / TR-369 원격 관리, GenieACS 구축
- 양산 검사 시스템 (Windows 툴링, DUT 자동 제어)
- 신뢰성 아키텍처: watchdog ladder, A/B failsafe boot

## 함께 일하기

본업과 병행하여 제한된 수의 컨설팅/외주 프로젝트를 진행합니다.
시간제가 아닌 **범위가 정해진 패키지** 방식이라, 결과물과 비용을
시작 전에 확정할 수 있습니다:

- **원격 디버깅 패키지** — 문제가 있는 장비나 회선을 가져오시면
  (모뎀 미접속, 스루풋 급락, 랜덤 리부트, "벤치에선 되는데 현장에선 안 되는"
  류의 문제) 원격 세션 2–3회를 진행하고, 근본 원인 분석과 수정 방안이 담긴
  보고서를 드립니다. 이 블로그의 글들이 다루는 문제가 바로 이 범위입니다.
- **TR-069 / ACS 구축** — GenieACS 기반 원격 관리 스택을 동작 가능한
  상태로 구축합니다: 프로비저닝 흐름, 펌웨어 업그레이드 경로, 운영을 위한
  런북까지.
- **신뢰성 / 복구 아키텍처 리뷰** — 부트 체인, watchdog 전략, 업그레이드
  경로를 설계 리뷰하고, 실제로 A/S 콜을 만들어낼 장애 모드의 우선순위와
  대응 방안을 정리해 드립니다.

위 항목에 딱 맞지 않는 문제라도 일단 연락 주세요. 맞지 않으면
맞는 방향을 알려드리겠습니다.

## 연락처

- 이메일: **pdh0085@gmail.com** <!-- TODO: replace with your public contact address -->
- GitHub: [pdh0085](https://github.com/pdh0085)

증상 요약과 하드웨어 정보(SoC, 모뎀 모델, OpenWrt/Yocto 버전)를
함께 보내주시면 가장 빠르게 유의미한 답변을 드릴 수 있습니다.
