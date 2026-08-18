---
title: "필드에서 절대 갱신되지 않는 코드"
date: 2026-08-17T10:00:00+09:00
draft: true
tags: ["openwrt", "uci-defaults", "overlayfs", "modemmanager", "hotplug", "embedded"]
summary: "새 overlay로 부팅할 때마다 약 185초 지점에서 시리얼 콘솔에 25바이트쯤 되는 바이너리가 찍혔다. 범인은 콘솔 UART를 모뎀으로 착각한 ModemManager였다. 필터를 만들어 uci-defaults에 넣었는데 아무 효과가 없었다 — 이유가 두 겹이었다."
---

<!-- Part 7 of the LTE572W Yocto-to-OpenWrt porting series -->

## 콘솔에 찍히던 네 바이트

새 overlay로 부팅할 때마다 약 185초 지점, 정확히 `init complete` 근처에서
시리얼 콘솔에 약 25바이트의 바이너리가 찍혔다. 그리고 곧 정상 출력이
돌아왔다. 재현은 매번 되는데 원인은 안 보였다.

이럴 때는 바이트를 읽으면 된다. 매번 같은 네 바이트가 들어 있었다.

```
... 00 78 f0 7e ...   ← HDLC: 명령 0x00 + CRC 78 f0 + 플래그 7e
```

HDLC 프레임 — 즉 QCDM/DIAG 프레임이다. 앞부분이 깨져 보이는 이유도 같이
설명된다. 누군가 포트를 열고 자기 프로브 baud로 `tcsetattr` 한 뒤(115200이
아니다) `AT`를 쓰고, 이어서 QCDM 프레임을 쓰고 닫은 것이다. AT 부분은 깨져
보이고 QCDM 부분은 그대로 찍힌다. **ModemManager가 콘솔 UART를 모뎀으로
착각하고 프로브하고 있었다.**

왜 콘솔까지 건드리나. 이 이미지의 MM은 udev 없이 빌드되어, 포트를 오직
`hotplug.d`가 부르는 `mmcli --report-kernel-event`로만 알게 된다. 그런데 순정
tty hotplug 스크립트는 **모든 tty를 보고한다** — 콘솔 UART(ttyS1)도, 심지어
가상 콘솔 `tty3`…`tty63`까지.

## 필터를 만들었다. 효과가 없었다

보고 필터를 써서 `uci-defaults`에 넣었다. 콘솔 쓰레기는 그대로였다. 이유가
두 겹이었고, 그 두 겹이 이 편의 전부다.

## 이유 1 — procd는 rcS보다 먼저 콜드플러그한다

```
커널 uevent 재생(STATE_EARLY) → [preinit: 필터가 있어야 할 곳]
    → rcS → S10boot(uci-defaults)   ← 여기서는 이미 늦다
    → S60 dbus → S70 ModemManager → 캐시 재생
    → (필터가 없으면) 콘솔 UART 프로브
```

커널 uevent는 procd의 `STATE_EARLY`에서 재생된다. `uci-defaults`(S10boot)든
어떤 init 스크립트든 그 시점엔 이미 늦다. 순정 스크립트가 벌써 `ttyS0`/`ttyS1`을
처리해 이벤트 캐시에 넣어 뒀고, MM은 시작하면서 그 캐시를 재생해 콘솔 UART를
프로브했다. procd가 exec 되기 전에 도는 곳은 preinit뿐이다. 그래서 답은
preinit이다.

## 이유 2 — uci-defaults는 이 보드에서 영원히 갱신되지 않는다

더 중요한 발견은 이쪽이다. `uci-defaults` 스크립트는 실행 후 `rm` 된다.
그런데 영구 overlay에서 `rm`은 삭제가 아니라 upper에 남는 **whiteout**이다.

```
/overlay/upper/etc/uci-defaults/
c---------  1 root  root  0, 0  ...  99-something
```

설정 유지 sysupgrade는 squashfs만 갈고 overlay는 그대로 두므로, 새 이미지가
가져온 `uci-defaults`는 이 whiteout에 **영원히 가려진다.** 실제 보드에서
확인하면 병합된 `/etc/uci-defaults`가 비어 있다. 순정 OpenWrt가 이 문제를 안
겪는 건 sysupgrade가 `rootfs_data`를 포맷하기 때문이고, 우리는 그러지 않는다.

버전 마커를 붙여서 풀 수 있는 문제도 아니다. **마커를 읽어야 할 스크립트
자체가 실행되지 않기 때문이다.** 그래서 일반 규칙 하나가 나온다 — 필드에서
갱신돼야 하는 로직은 `uci-defaults`에 두면 안 된다. `/lib/preinit`과
`/etc/init.d`는 squashfs 파일이라 whiteout되지 않고 항상 새 이미지에서 온다.

## 제대로 된 필터의 모양

- **tty는 화이트리스트**: `ttyACM*`, `ttyUSB*`, `cdc-wdm*`. 우리가 파는 모든
  모뎀을 덮는다.
- **net은 블랙리스트**: 모뎀의 네트워크 인터페이스 이름은 예측할 수 없다
  (`wwan0`/`usb0`/…). 화이트리스트로 만들면 언젠가 WAN이 조용히 죽는다.
- **no-bus 가드**: 모뎀은 S11에서 전원이 들어와 약 50초에 열거되는데 MM은
  S70이다. 즉 대부분의 포트 이벤트는 MM이 없는 시점에 발생하고, 그때 `mmcli`를
  불러 봐야 6~13초를 쓰고 `couldn't get bus`로 실패할 뿐이다. 그래서 이벤트만
  캐시하고 포크를 건너뛴다. MM 래퍼가 시작할 때 캐시를 재생하므로 정보는 잃지
  않는다 — 실제로 MM보다 약 90초 먼저 나타난 모뎀을 `mmcli -L`이 정상 인식했다.

결과만 적으면 부팅 중 `mmcli` 포크가 23회에서 0회가 됐다. 이게 부팅 시간에
어떤 의미인지는 다음 편에서 수치로 다룬다. 그리고 이 가드에도 함정이 하나
숨어 있었다 — `cdc-wdm` 제어 노드는 자기 hotplug 이벤트가 없어서 초기 버전
가드가 그 보고를 삼켰고, 모뎀이 조용히 AT+PPP로 강등된 적이 있다. 그 이야기는
이 시리즈의 셀룰러 편에서 한다.

## 덤으로 배운 함정 — 마커 문자열의 부분 일치

이 hotplug 스크립트의 사본은 `/etc`, 즉 영구 overlay에 놓인다. 옛 사본이
펌웨어 업데이트보다 오래 산다는 뜻이다. 그래서 "이미 패치됨?" 마커를 **버전
있는 문자열**(`QUARK-MM-HOTPLUG-vN`)로 두고 **앵커된 정확 일치**로 검사한다.
부분 문자열 검사(`grep -q QUARK-MM-HOTPLUG`)는 `-v1`이나 `-OLD`에도 매칭된다.
실제로 낡은 사본 하나가 그렇게 "업데이트"를 통과해 살아남았고, 그걸 찾는 데
시간을 꽤 썼다.

## 배운 것

- 스크립트를 놓을 자리를 고를 때 "언제 실행되는가"만큼 **"업데이트되면 다시
  실행되는가"** 를 물어야 한다. overlay 기반 시스템에서 삭제는 삭제가 아니라
  가림이다.
- 자동 탐색(probe)은 편리한 만큼 위험하다. 대상이 콘솔일 수도 있다. 나중에
  우리 시리얼 데몬도 같은 이유로 콘솔 장치 열기를 거부하게 만들었다.
- 정체불명의 바이트 덩어리는 읽으면 된다. 프로토콜 프레임은 대개 자기가
  누구인지 말해준다.
