---
title: "아무도 읽지 않는 설정 파일 — FRR을 UCI로"
date: 2026-08-17T10:00:00+09:00
draft: true
tags: ["openwrt", "frr", "uci", "ospf", "procd", "quark-x1000"]
summary: "웹 UI에서 OSPF를 저장하면 에러 없이 저장됐다. 그런데 라우팅 데몬은 아무것도 몰랐다 — 페이지가 쓰던 파일을 아무도 읽지 않았기 때문이다. UCI 생성기로 소유권을 다시 세우는 이야기와, 분리 실행한 재시작이 fd 1000을 물고 늘어져 이후 모든 start/reload를 영원히 멈추게 한 함정에 대하여."
---

<!-- Part 11 of the LTE572W Yocto-to-OpenWrt porting series -->

## 증상 — 저장은 되는데 아무 일도 없다

이 제품에는 라우팅 페이지가 있다. RIP, OSPF, BGP. 저장을 누르면 에러 없이
저장된다. 그런데 `vtysh`로 들어가 보면 라우팅 데몬은 아무것도 모른다.

세 단계로 쪼개서 봤다. UI에서 저장 → 파일 내용 확인 → 실제 running-config
확인. 1단계와 2단계는 성공했고, 3단계만 비어 있었다. 저장 성공은 도달을
증명하지 않는다.

## 원인 — frr.conf가 존재하는 순간

페이지들은 `/etc/frr/{ospfd,ripd,bgpd}.conf`에 쓰고 있었다. 문제는 FRR의
시작 스크립트 `frrcommon.sh`가 **`/etc/frr/frr.conf`가 존재하는 순간 통합
설정(integrated-config) 모드로 전환**한다는 것이다.

```sh
daemon_prep(): [ -r "$C_PATH/frr.conf" ] && return 0  # 데몬별 .conf는 준비도, 사용도 안 한다
vtysh_b():     [ -r "$C_PATH/frr.conf" ] || return 0  # frr.conf를 모든 데몬에 밀어 넣는다
```

그리고 이 보드는 그 파일을 출하 시부터 갖고 있었다. 즉 페이지는 몇 년째
읽히지 않는 파일에 성실하게 쓰고 있었던 것이다. OSPF가 한 번도 end-to-end로
증명된 적이 없었던 이유는 데몬이 아니다. 글이 허공에 쓰이고 있었다.

## 새 구조 — 파일의 주인을 정한다

수정의 핵심은 필드 몇 개를 고치는 게 아니라 소유권을 정하는 것이었다.

```
/etc/config/frr (UCI) → frr-gen → /etc/frr/frr.conf → FRR
                           │
                           └─ vtysh -C 드라이런 게이트
```

웹 페이지는 raw 파일 편집기가 아니라 UCI 위의 구조화된 폼이 되고, 생성기
`frr-gen`이 `frr.conf` 자체를 렌더링한다(init 스크립트는 START=94, FRR의
S95 바로 앞). 설계 결정은 다섯 개였다.

- **설치 전에 `vtysh -C`로 드라이런 파싱.** 문법 오류 하나가 다음 재시작에서
  라우팅 스택 전체를 내려버린다. 거부되면 이전 설정이 계속 돌고, 페이지가 그
  사실을 알린다.
- **종료 코드를 계약으로.** `0` = 새 설정 설치됨(재시작), `2` = 변경
  없음(재시작 금지 — 의미 없는 재시작은 인접성을 끊는다), `1` = 거부됨.
  "변경 없음"을 구분하지 않으면 저장 버튼을 누를 때마다 라우팅이 흔들린다.
- **탈출구를 남긴다.** 스키마가 모델링하지 못하는 FRR 문법은 섹션별 `raw`
  리스트로 그대로 출력한다. FRR의 설정 표면은 어떤 스키마보다 넓어서, 이게
  없으면 UCI 프론트엔드는 기능 후퇴다.
- **소유권을 명시한다.** UCI가 `frr.conf`의 단일 소유자다.
  `frr.global.managed='0'` 한 줄로 소유권을 사람에게 돌려줄 수 있고, 그러면
  페이지는 "수동 모드" 배너를 띄운다. 주인이 아닌 상태에서 주인인 척하지
  않는 것.
- **기존 파일을 지우지 않는다.** 손으로 쓴 `frr.conf`가 이미 있으면 통째로
  `frr.global.raw`로 임포트하고 원본은 `frr.conf.pre-uci`로 보관한다.

## 함정 — flock 1000

`frr restart`는 이 보드에서 수십 초가 걸린다. rpcd의 exec 타임아웃을 훌쩍
넘겨서 LuCI의 Save & Apply가 XHR 타임아웃으로 죽는다. 그래서 재시작을 분리
실행했더니, 이후 모든 `start`/`reload`가 영원히 멈췄다.

원인은 procd였다. `procd.sh`는 서비스별 락을 `exec 1000>...; flock 1000`으로
잡는데 **fd 1000에 close-on-exec가 없다.** 분리 실행한 프로세스가 그 fd를
상속하고, 그게 다시 watchfrr/mgmtd/zebra/ospfd에 상속된다. 라우팅 데몬들이
FRR이 도는 내내 우리 서비스의 락을 쥐고 있는 것이다. 해결은 스폰할 때 그
fd를 닫는 것뿐이다.

```sh
( exec 1000>&- ; setsid /etc/init.d/frr restart </dev/null >/dev/null 2>&1 & ) &
```

같은 함정을 Wi-Fi 재적용 스크립트에서도 만났다. 이 시스템에서는 "분리 실행 =
fd 정리 필요"가 규칙이 됐다.

## 검증 — 끝단에서 확인한다

UI에서 OSPF를 켜면 uci → 생성기 → `frr.conf`를 거쳐 running-config에
`router ospf`와 router-id, 네트워크가 뜬다.

```
vtysh -c 'show running-config' | grep -A5 'router ospf'
vtysh -c 'show ip ospf interface br-lan'
```

`show ip ospf interface`에서 `br-lan`이 area 0에 `No Hellos (Passive
interface)`로 보인다. **이 보드에서 UI로 설정한 라우팅이 FRR에 실제로 도달한
첫 사례였다.** 작은 디테일 하나 — FRR 10에서 `passive-interface X`는
deprecated(VRF를 인식하지 않는다)라 생성기가 인터페이스별 `ip ospf passive`
형태로 바꿔 출력한다. 생성한 파일과 running-config가 글자 그대로 일치해야
다음 사람이 어느 쪽을 믿을지 고민하지 않는다.

## 배운 것

- 설정 UI를 만들 때 첫 질문은 "어떤 필드를 노출할까"가 아니라 **"이 파일의
  주인은 누구인가"** 다.
- 생성기는 검증(드라이런)과 무변경 감지가 없으면 위험한 도구다.
- 스키마 기반 UI에는 반드시 탈출구가 필요하다. 없으면 사용자는 UI를
  우회하고, 그 순간 소유권 계약이 깨진다.
- "저장됨"은 "적용됨"이 아니고, "적용됨"은 "동작함"이 아니다. 확인은 항상
  끝단에서.
