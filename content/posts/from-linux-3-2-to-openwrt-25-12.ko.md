---
title: "리눅스 3.2에서 OpenWrt 25.12로: MA-2080X 셀룰러 라우터 포팅기"
date: 2026-10-29T09:44:43+09:00
draft: false
tags: [openwrt, embedded, am335x, u-boot, firmware-update, reliability, debugging, ucode]
summary: "산업용 셀룰러 라우터 MA-2080X를 벤더 리눅스 3.2 펌웨어에서 OpenWrt 25.12.5로 옮긴 약 7주의 기록. 되돌릴 수 없는 변경은 한 번에 하나씩, 벤더 펌웨어 기능 따라잡기, 조용히 실패한 버그들, 그리고 이 결과물이 출시되지 않는 이유."
aliases: ["/ko/posts/openwrt-25-12-on-a-legacy-am335x-router/"]
---

## 왜 했나

2026년 가을 약 7주 동안 Maxon / RF Industries의 셀룰러 라우터 MA-2080X를 벤더의 리눅스 3.2 펌웨어에서 OpenWrt 25.12.5로 옮겼다. 그 결과물인 OS-DHP 2.2.6은 벤치 장비 두 대에서 A/B 업데이트, 웹 UI, TR-069 / TR-369 관리, 그리고 벤더 펌웨어가 가진 기능 거의 전부를 갖추고 돌고 있다.

MA-2080X는 모듈식 산업용 라우터다. TI AM3352 보드, eMMC, Sierra Wireless 셀룰러 모듈, SIM 슬롯 두 개, 시리얼 포트, 디지털 I/O, GNSS로 이뤄져 있다. 기본 펌웨어는 아직도 십 년도 더 된 3.2 커널로 돌았다. 최신 보안 수정이 없고, TLS 스택이 낡았고, 요즘 관리 프로토콜을 붙일 현실적인 방법이 없다는 뜻이다.

이 펌웨어를 실은 제품은 나오지 않는다. 그래서 이 글은 기록이다. 포팅에 무엇이 들었는지, 무엇에 물렸는지, 다시 해도 똑같이 할 것은 무엇인지.

## 출발점

2015년 회로도, 벤더의 리눅스 3.2 소스 트리, 벤치 장비 두 대로 시작했다. 회로도와 벤더 보드 파일이 다르면 보드 파일을 따랐다. 실제로 출하된 것은 보드 파일 쪽이기 때문이다.

하드웨어는 이렇게 생겼다.

- **SoC와 메모리:** TI AM3352(Cortex-A8), 512 MB RAM, 저장장치는 eMMC 하나뿐이다. GPMC 핀은 전부 eMMC 데이터, 두 번째 RGMII 포트, GPIO 중 하나로 재사용된다.
- **이더넷:** 스위치가 아니라 독립된 EMAC 두 개다. OpenWrt의 omap 타깃은 이미 am335x 보드를 레거시 `cpsw` 드라이버로 되돌려 쓰고 있어서, 디바이스 트리에는 평범한 MAC 포트 두 개를 넣었다.
- **셀룰러:** 확장 소켓에 Sierra Wireless 모듈, 먹스 뒤에 SIM 슬롯 두 개가 있고, SIM 유무는 ADC 전압 구간으로 읽는다. 양산 보드에서 GNSS는 모듈에서 나온다.
- **현장 I/O:** TCP/시리얼 게이트웨이로 쓰는 시리얼 포트, 디지털 입출력, 아날로그 입력, 상태 LED, 리셋 버튼.

eMMC에는 벤더 파티션이 들어 있다. p1/p2에 MLO와 U-Boot, p3에 벤더 "nvram" 설정 저장소, p4에 U-Boot 환경, p5/p6에 커널 슬롯 두 개, p7에 미디어 파티션이다.

벤더 펌웨어는 전부 RAM에서 돈다(`root=/dev/ram0`). 안드로이드 부트 이미지를 16비트 체크섬이 붙은 작은 벤더 컨테이너로 감싼 형태다. 이 컨테이너를 리버스 엔지니어링해서 바이트 단위로 똑같이 만들어 냈다. 벤더의 패커는 헤더 필드 두 개를 초기화하지 않은 스택에서 써 넣고 있어서, 돌릴 때마다 그 값이 달랐다.

## 브링업: OpenWrt 먼저, 부트로더는 나중에

계획은 되돌릴 수 없는 것을 한 번에 하나만 바꾸는 것이었다. 그래야 현장 장비에 사람이 갈 일이 생기지 않는다.

1. **1단계, 벤더 부트로더 위의 OpenWrt.** 벤더 U-Boot(2011.09)는 로드 주소와 커맨드 라인이 고정된 안드로이드 부트 이미지를 부팅한다. `target/linux/omap`용 OpenWrt를 커널과 initramfs로 정확히 그 형식에 맞춰 빌드했다. 그래서 첫 이전은 벤더 자신의 웹 페이지로 한 번 업로드하는 것으로 끝난다. 이 보드에서 OpenWrt가 처음 부팅한 날은 9월 11일이다.
2. **2단계, 벤치에서 새 U-Boot.** 벤더 부트로더에는 폴백이 아예 없었다. 이미지가 잘못되면 프롬프트에서 영원히 멈췄고, SPL은 워치독을 꺼 버렸다. U-Boot 2025.10을 이 보드로 포팅했다. 처음에는 벤더 프롬프트에서 체인로딩했고, 다음에는 맨 패드 두 개를 브리지해 부팅 순서를 바꾼 뒤 microSD 카드로 부팅했다. eMMC의 MLO에 손대기 전에, 이 SD 경로가 검증된 벽돌 복구 경로가 됐다.
3. **3단계, 웹 UI에서 부트로더 교체.** OS 이미지마다 짝이 맞는 MLO / u-boot.img가 들어 있다. 설치 작업은 보드, 파티션 테이블, 현재 체크섬을 확인하고, p2 다음 p1 순서로 쓰고, 각각 다시 읽어 보고, 맞지 않으면 이전 것을 복원한다.

새 U-Boot가 들어가면 커널 슬롯 두 개가 진짜 A/B 쌍이 된다. sysupgrade는 비활성 슬롯에 쓰고 검증한 뒤에야 `ab_primary`를 바꾼다. U-Boot는 부팅 시도 횟수를 세고, 세 번 실패하면 다른 슬롯으로 부팅한다. 커널이 멈추면 하드웨어 워치독이 그것을 실패 한 번으로 만든다. 사용자 공간이 정상적으로 올라온 부팅은 카운터를 지운다.

<figure style="margin:1.5em 0">
<svg viewBox="0 0 760 484" role="img" aria-label="잘못된 이미지는 알아서 다른 슬롯으로 돌아간다" font-size="13" style="width:100%;height:auto;font-family:inherit">
<defs><marker id="ab-arrow-ko" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M0 0L10 5L0 10z" fill="var(--secondary)"/></marker></defs>
<text x="24" y="34" font-size="15" font-weight="600" fill="var(--primary)">잘못된 이미지는 알아서 다른 슬롯으로 돌아간다</text>
<g fill="none" stroke="var(--secondary)" stroke-width="1.25"><path d="M208 100H288" marker-end="url(#ab-arrow-ko)"/><path d="M472 100H544" marker-end="url(#ab-arrow-ko)"/><path d="M636 126V174" marker-end="url(#ab-arrow-ko)"/><path d="M552 210H472" marker-end="url(#ab-arrow-ko)"/><path d="M636 246V294" marker-end="url(#ab-arrow-ko)"/><path d="M380 236V284" marker-end="url(#ab-arrow-ko)"/><path d="M544 320H464" marker-end="url(#ab-arrow-ko)"/><path d="M296 320H208" marker-end="url(#ab-arrow-ko)"/><path d="M380 356V404" marker-end="url(#ab-arrow-ko)"/><path d="M472 430H744V100H728" marker-end="url(#ab-arrow-ko)"/></g>
<rect x="24" y="74" width="184" height="52" rx="8" fill="none" stroke="var(--secondary)" stroke-width="1.25"/><text x="116" y="96" text-anchor="middle" fill="var(--primary)">예비 슬롯에 쓰고</text><text x="116" y="112" text-anchor="middle" fill="var(--primary)">검증</text>
<rect x="288" y="74" width="184" height="52" rx="8" fill="none" stroke="var(--secondary)" stroke-width="1.25"/><text x="380" y="96" text-anchor="middle" fill="var(--primary)">ab_primary 전환,</text><text x="380" y="112" text-anchor="middle" fill="var(--primary)">부팅 카운터 활성화</text>
<rect x="544" y="74" width="184" height="52" rx="8" fill="none" stroke="var(--secondary)" stroke-width="1.25"/><text x="636" y="96" text-anchor="middle" fill="var(--primary)">U-Boot가 시도</text><text x="636" y="112" text-anchor="middle" fill="var(--primary)">횟수를 하나 올림</text>
<polygon points="552,210 636,174 720,210 636,246" fill="none" stroke="var(--secondary)" stroke-width="1.25"/><text x="636" y="214" text-anchor="middle" fill="var(--primary)">3회 초과?</text>
<rect x="288" y="184" width="184" height="52" rx="8" fill="none" stroke="var(--secondary)" stroke-width="1.25"/><text x="380" y="214" text-anchor="middle" fill="var(--primary)">ab_primary로 부팅</text>
<rect x="544" y="294" width="184" height="52" rx="8" fill="none" stroke="var(--secondary)" stroke-width="1.25"/><text x="636" y="324" text-anchor="middle" fill="var(--primary)">다른 슬롯으로 부팅</text>
<polygon points="296,320 380,284 464,320 380,356" fill="none" stroke="var(--secondary)" stroke-width="1.25"/><text x="380" y="324" text-anchor="middle" fill="var(--primary)">사용자 공간 정상?</text>
<rect x="24" y="294" width="184" height="52" rx="8" fill="#3b82f6" fill-opacity="0.12" stroke="#3b82f6" stroke-width="2"/><text x="116" y="316" text-anchor="middle" font-weight="600" fill="var(--primary)">부팅 확인,</text><text x="116" y="332" text-anchor="middle" fill="var(--primary)">카운터 초기화</text>
<rect x="288" y="404" width="184" height="52" rx="8" fill="none" stroke="var(--secondary)" stroke-width="1.25"/><text x="380" y="426" text-anchor="middle" fill="var(--primary)">워치독이</text><text x="380" y="442" text-anchor="middle" fill="var(--primary)">보드를 리셋</text>
<text x="512" y="202" text-anchor="middle" font-size="11.5" fill="var(--secondary)">아니오</text>
<text x="646" y="274" font-size="11.5" fill="var(--secondary)">예</text>
<text x="252" y="312" text-anchor="middle" font-size="11.5" fill="var(--secondary)">예</text>
<text x="390" y="384" font-size="11.5" fill="var(--secondary)">아니오, 멈춤</text>
<text x="608" y="422" text-anchor="middle" font-size="11.5" fill="var(--secondary)">리셋 후 다음 시도</text>
</svg>
<figcaption style="text-align:center;font-size:0.85em;font-weight:400;color:var(--secondary)">A/B 업데이트와 폴백 · bootlimit 3인 U-Boot 부팅 카운터</figcaption>
</figure>

잘못된 업그레이드와 정상 동작하는 장비 사이에 사람이 낄 필요가 없다. 멈춤은 워치독 리셋이 되고, 세 번 실패하면 U-Boot가 전에 잘 돌던 슬롯으로 부팅한다.

RAM 루트 때문에 생긴 문제가 하나 더 있었다. OpenWrt의 일반적인 overlay는 `pivot_root`가 필요한데, 커널은 initramfs에서 이를 거부한다. 그래서 preinit 훅이 새 ext4 `config` 파티션을 마운트하고 `/etc`만 overlay로 덮는다. 벤더 펌웨어에서 막 넘어온 장비에서는 그보다 앞선 훅이 공장 파티션 테이블을 체크섬으로 알아보고, 미디어 파티션을 줄이고, 미리 계산해 둔 바이트로 새 GPT를 쓴다. 그다음 가져오기 도구가 벤더의 p3 설정 저장소를 읽는다(아래에서 다시 다룬다).

실제 장비에 처음 부트로더를 설치했을 때도 고약한 버그가 하나 나왔다. 쓰기와 다시 읽기 검증은 통과했는데, 장비가 다시는 자동 부팅하지 않았다. 손으로 고친 패치의 hunk 헤더가 틀려서 부팅 변수 하나가 정의되지 않았기 때문이다. 시리얼 콘솔과 신중한 쓰기 순서 덕분에 죽은 장비가 아니라 고칠 거리로 끝났다.

## 벤더 펌웨어 기능 따라잡기

라우터가 제 일을 못 하면 새 기반 시스템은 운영자에게 아무 가치가 없다. 그래서 브링업 이후 시간 대부분은 벤더 펌웨어에 있던 기능을 만드는 데 들어갔다. 다른 제품에도 쓰는 공용 OpenWrt 애플리케이션 프레임워크인 dhplab 위에 만들었다. 10월에 벤더 소스를 훑어 OS-DHP에 아직 없는 것을 찾았고, 기능 브랜치 14개로 그 차이를 메웠다. 브랜치마다 리뷰를 거친 뒤에 병합했다.

| 영역 | OS-DHP가 하는 일 |
| --- | --- |
| 셀룰러 | 페일오버가 되는 듀얼 SIM(신호 없음, 연결 실패, 로밍, 데이터 한도), SIM별 APN, 이더넷과 셀룰러 사이 업링크 전환, SIM별 데이터 사용량, 셀룰러 망에서 받는 네트워크 시간 |
| 시리얼 | 벤더의 AT 명령 "IPStack" 세트를 갖춘 TCP / UDP 시리얼 게이트웨이, 다이얼 에뮬레이션, AT over IP, 연결을 끊는 대신 역압(back-pressure) |
| 산업용 프로토콜 | Modbus(벤더의 FTP 업로드 형식 포함), 다이얼아웃과 비요청 보고를 지원하는 DNP3 아웃스테이션 |
| I/O와 알람 | 디지털·아날로그 I/O, SMS와 이메일 그룹으로 가는 알람 이벤트, SMS 명령(상태, SIM 전환, APN, 재부팅 등), 이벤트 시 사용자 스크립트 |
| GNSS | 셀룰러 모듈에서 받는 위치, 서버로 보내는 NMEA·추적 보고, 지오펜스 |
| 네트워킹 | IPsec(IKEv1 aggressive 모드 포함), 인증서 저장소가 있는 OpenVPN, PPTP, VRRP, 벤더 MIB를 쓰는 SNMP, 콘텐츠 필터링, 예약 재부팅, 절전 |
| 관리 | 웹 UI, TR-069, WebSocket·STOMP 위의 TR-369 / USP, RADIUS 로그인, 로그 내보내기, 서명된 플러그인 |

이것을 재설치가 아니라 이전으로 만드는 것은 설정 가져오기다. 첫 부팅 때 벤더 설정을 읽어 대응하는 OpenWrt 설정을 쓴다. 그래서 업로드 후 장비는 LAN, APN, 시리얼 포트, I/O, 알람 연락처가 예전 그대로인 상태로 올라온다. 새 펌웨어용 236쪽짜리 사용자 매뉴얼도 이 작업에 포함됐다.

## 물린 것들

날린 날 대부분은 조용히 실패한 것들 때문이었다. 남겨 둘 만한 것만 적는다.

**하드웨어**

- **신호 이름은 거짓말을 한다.** 여러 GPIO(SIM 먹스, 모뎀 전원, RF 활성화)의 극성이 이름과 반대였다. 맞는 쪽은 벤더 코드와 측정이었고, 회로도의 넷 이름은 틀렸다.
- **부품 하나가 버스 전체를 막을 수 있다.** 개발 보드에서는 개별 GNSS 수신기가 I2C0를 붙잡고 있어서, 그 뒤의 전원 관리 IC에 닿을 수 없었다. I2C 사용자들이 probe를 마칠 때까지 수신기를 리셋 상태로 묶어 두고, PMIC가 끝내 나타나지 않으면 CPU 주파수 조절이 스스로 꺼지게(1 GHz 대신 720 MHz) 해서 고쳤다.
- **아날로그 문턱값은 보드 리비전에 따라 다르다.** SIM 유무는 ADC 전압 구간이고, 그 구간은 PCB 리비전마다 다르다. 리비전 설정이 틀려도 오류는 없다. 답이 틀릴 뿐이다.

**소프트웨어**

- **ucode는 기본적으로 정규식을 REG_NEWLINE으로 컴파일한다.** ucode 2026.04.24에서는 정규식에 `s` 플래그가 없으면 `^`와 `$`가 줄마다 일치한다. 그래서 앵커를 건 검증기가 `"ok\nevil"`을 통과시킨다. 그렇게 들어온 줄바꿈이 셸 명령줄까지 넘어갔다. 감사를 해서 프레임워크와 제품 계층의 검증기·정화 함수를 전부 고쳤다.
- **기본 sysupgrade는 RAM 루트에서 백업을 거부하면서도 0으로 끝난다.** 이 보드의 루트는 항상 램디스크라서, 2.2.4 전까지 모든 빌드에서 "설정 다운로드"가 빈 파일을 내놓았다. 아무것도 오류를 보고하지 않았고, 복원 시험에서 드러났다.
- **논블로킹 소켓은 믿을 만한 소켓이 아니다.** 시리얼 게이트웨이는 `EAGAIN`에서 TCP 연결을 끊었다. 상대가 잠시 멈추면 약 45 KB 뒤에 연결이 닫혔고, 나머지 스트림은 AT 명령으로 해석됐다. 크기가 정해진 큐와 UART 역압으로 고쳤다.
- **벤더에는 따라 하지 않을 만한 버그가 있었다.** 지오펜스는 각도 형식을 섞어 써서 한 번도 동작하지 않았고, 절전 설정은 시리얼 트랜시버를 루프백으로 만들었고, DNP3 클래스 마스크는 일대일로 대응하지 않았다. 가져오기 도구는 이것들을 그대로 옮기지 않고 변환한다.
- **빌드 시스템은 말없이 어긋날 수 있다.** 로그로 파이프한 빌드 스크립트에서는 `./scripts/feeds update`가 "failed."를 출력해도 종료 코드가 0으로 보였고, 피드를 바꾼 뒤의 `make defconfig`가 벤더 접두사 옵션을 초기화한 적도 있다. 지금은 빌드 가드가 `.config`나 피드 고정값이 기록과 다른 빌드를 거부한다.
- **overlay는 잊지 않는다.** 손으로 `/etc`에 복사한 파일은 영구 저장소로 올라가서, 이미지에 든 같은 파일을 영원히 가린다.

## 작업을 어떻게 굴렸나

속도보다 규율이 중요했다. 현장 장비에서 실수하면 그때마다 출장이 생기기 때문이다.

- **서로 다른 벤치 장비 두 대.** 한 대는 PMIC 버스가 죽어 720 MHz로 돌았고, 다른 한 대는 정상인 구형 보드 리비전으로 1 GHz로 돌았다. 모든 변경을 두 대에서 돌렸고, 쓰기 전에는 MAC 주소로 장비를 확인했다. 시리얼 콘솔은 늘 붙여 두었고, 장비 안의 작은 검사기가 6시간마다 장시간 시험을 돌렸다.
- **빌드 하나하나가 기록이다.** 래퍼가 깨끗한 빌드마다 번호를 매기고, 이미지를 체크섬·고정값과 함께 백업하고, 설정이나 피드 고정값이 어긋나면 빌드를 거부한다. 80개가 넘는 빌드마다 무엇이 바뀌었고 장비에서 무엇을 확인했는지 적은 릴리즈 노트 항목이 있다.
- **플래시 절차는 하나.** 체크섬, 설정 백업, 업로드, 장비에서 검증, 업그레이드, 그다음 슬롯, 부팅 카운터, 설정 저장소, 부트 로그, 클라우드 연결 확인. 항상 백업해 둔 이미지로 하고, 빌드 디렉터리에서 바로 하지 않는다.
- **진짜 관리 서버.** 시험 서버에 TR-069, TR-369(USP), XMPP, STUN을 돌려서, 원격 설정·백업·복원을 가정하지 않고 처음부터 끝까지 시험했다.
- **AI의 도움, 결정은 사람.** 작업은 Claude Code와 함께 했다. 리드 세션이 계획하고 검토했고, 워커 에이전트가 코드를 쓰고 벤치 확인을 돌렸고, 공용 프레임워크를 맡은 별도 세션이 모든 브랜치를 리뷰했다. 빌드, 플래시, 부트로더 쓰기, 그리고 모든 제품 결정은 내 말을 기다렸다.

## 어디서 끝났나

10월 27일에 빌드한 OS-DHP 2.2.6이 최종 릴리스다. 벤치 장비 두 대에서 모든 검사를 통과하며 돌고 있고, 236쪽 사용자 매뉴얼, 4쪽 개요서, 릴리즈 노트가 함께 있다.

MA-2080X를 판매하는 회사에 이 펌웨어를 제안했고, 회사는 출시하지 않기로 했다. 포팅이 아니라 제품에 대한 결정이었고, 나도 이해한다.

잃은 작업은 많지 않다. 위 기능 대부분은 공용 dhplab 프레임워크에 만들었고, 이미 다른 제품들이 쓰고 있다. SMS 명령, 시리얼 게이트웨이의 역압, 서명된 플러그인, 보안 수정, ucode 줄바꿈 감사가 그렇다. 보드 포팅, U-Boot 패치, 빌드 기록은 보관해 두었다. 언젠가 필요해지면 바로 꺼낼 수 있다.

## 다시 해도 똑같이 할 것

1. **위험한 쓰기 전에 복구 경로부터 동작하게 만든다.** MLO에 손대기 전에 SD 카드 부팅을 검증해 두었고, 그 덕분에 부트로더 작업을 차분하게 할 수 있었다.
2. **단계마다 되돌릴 수 없는 변경은 하나만.** 먼저 기존 부트로더 위에 OpenWrt, 새 부트로더는 나중에 따로. 둘을 한 번의 무인 부팅에 같이 넣지 않는다.
3. **"exit 0"은 결과가 아니라 주장으로 다룬다.** 빈 백업, 조용한 피드 실패, 정규식 앵커는 반환값만 보는 검사를 전부 통과했다. 결과를 시험해야 한다. 백업을 복원해 보고, 설정을 diff해 보고, 바이트를 실제로 보내 본다.
4. **출하된 것과 측정한 것을 믿는다.** 회로도보다 벤더 보드 파일, 넷 이름보다 측정.
5. **하면서 적는다.** 빌드마다 릴리즈 노트, 발견마다 엔지니어링 노트. 이 글도 그 기록으로 썼다.

MA-2080X는 캐비닛으로 들어간다. 그것이 가르쳐 준 것들은 그렇지 않다.
