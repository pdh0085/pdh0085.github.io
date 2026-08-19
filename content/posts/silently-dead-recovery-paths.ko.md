---
title: "조용히 죽어 있던 복구 경로 두 개"
date: 2026-08-17T10:00:00+09:00
draft: true
tags: [openwrt, sysupgrade, factory-reset, procd, shell-scripting, embedded]
summary: "`sysupgrade -n`은 한 번도 설정을 지운 적이 없었고, 공장초기화는 서로 다른 이유로 두 번 죽어 있었다. 둘 다 에러 없이 성공했다고 보고했다. 하나는 프로세스 경계에서 변수가 죽었고, 하나는 볼륨 추상화가 256 KB 정렬 패딩을 rootfs_data라고 자신 있게 선언한 탓이었다."
---

<!-- Part 6 of the LTE572W Yocto-to-OpenWrt porting series -->

복구 기능에는 고약한 성질이 하나 있다. 평소에는 아무도 쓰지 않기 때문에,
고장 나도 아무도 모른다는 것이다. 이번 편은 LTE572W에서 그렇게 조용히 죽어
있던 경로 두 개를 찾은 기록이다. 두 기능 모두 에러 없이 "성공"하면서
아무것도 하지 않고 있었다. 고장 난 이유도 서로 달랐는데, 하나는 프로세스
경계였고 하나는 추상화의 오작동이었다.

## 1. `sysupgrade -n`이 설정을 지우지 않는다

증상. "설정 초기화" 옵션으로 펌웨어를 새로 올렸는데 예전 설정이 그대로다.
더 나쁜 건, overlay가 살아남으니 `uci-defaults`도 다시 실행되지 않는다는
것이다. "공장 초기화된 새 펌웨어"가 전혀 새것이 아니었다.

코드를 보면 멀쩡해 보인다. `platform_do_upgrade` 안에 이런 판단이 있었다.

```sh
[ "${SAVE_CONFIG:-1}" != "1" ] && rm -f "$overlay_img"
```

문법도 맞고 논리도 맞다. 문제는 이 변수가 그 지점까지 오지 못한다는 것이다.

`platform_do_upgrade`는 stage 2에서 실행된다. `ubus call system sysupgrade`
이후 procd가 새 환경으로 exec 하기 때문에, 그 경계를 넘는 값은
prefix / path / force / backup / command / options 뿐이고, procd는 그것들을
`$IMAGE`, `$UPGRADE_BACKUP`, `$WDTFD`, `$UPGRADE_OPT_*`로 바꿔 넘긴다.

```
stage1 (/sbin/sysupgrade)
   SAVE_CONFIG=1 export ─────────┐
   ubus call system sysupgrade   │  이 경계를 넘는 것:
                                 │  prefix / path / force / backup / command / options
                                 ▼
stage2 (procd가 새 환경으로 exec)
   $IMAGE  $UPGRADE_BACKUP  $WDTFD  $UPGRADE_OPT_*
   ← $SAVE_CONFIG는 여기 없다 (빈 값 → "설정 유지"로 기본 동작)
```

`$SAVE_CONFIG`는 stage 1에서 export되고 거기서 죽는다. 그래서 저 테스트는
항상 빈 변수를 읽고, `:-1` 기본값에 걸려 "설정 유지"로 동작한다. `-n`을
줬든 안 줬든, 언제나.

올바른 신호는 이미 경계를 건너오고 있었다. stage 1의 `sbin/sysupgrade`는
`SAVE_CONFIG=1`일 때만 ubus 호출에 `backup`을 넣는다. 그러니까
`$UPGRADE_BACKUP`이 없다는 것 자체가 초기화 신호인 셈이다. 조건을 그걸로
바꾸고 시리얼로 양방향 확인했다. `-n`이면 overlay 제거 로그가 찍히고 심어
둔 마커(APN, 호스트명)가 사라지고, 평소 업그레이드면 살아남는다.

교훈은 한 줄이다. 셸의 `${VAR:-기본값}`은 편리하지만, "변수가 없다"와
"변수가 기본값이다"를 구분하지 않는다. 프로세스 경계를 넘는 코드에서는
위험하다.

## 2. 공장초기화가 두 번, 서로 다른 이유로 죽어 있었다

배경부터. 이 보드에서는 `factoryreset` 바이너리 하나가 firstboot, 리셋
버튼, TR-069의 `rpc-sys factory`를 전부 담당한다. 하나를 고치면 셋이 같이
고쳐지고, 하나가 망가지면 셋이 같이 망가진다.

1차 고장은 볼륨을 못 찾는 것이었다. 순정 코드는 `volume_find("rootfs_data")`로
시작한다. 진짜 플래시 파티션이 있는 보드라면 당연한 가정인데, 우리 overlay는
FAT 위의 loop-ext4 이미지 파일이라 어느 드라이버도 못 맞춘다.
mtd(`/proc/mtd` 자체가 없다)도, ubi도, partname도.

```
factoryreset: MTD partition 'rootfs_data' not found
```

그리고 중단. 모든 초기화 경로가 조용히 아무 일도 하지 않았다. 여기까지는
우리 배치가 특이한 탓이지 코드 잘못이 아니다.

1차 수정을 했고, 그게 나중에 썩는다. "볼륨을 못 찾으면 마운트된
`/overlay`를 지운다"는 폴백을 넣었다. 잘 됐다. 루트가 squashfs loop로
바뀌기 전까지는.

2차 고장은 추상화가 "성공"하는 경우였다. 루트가 squashfs가 되자 `rootdisk`
드라이버가 붙었다. 이 드라이버는 루트 블록 장치에서 squashfs 슈퍼블록을
찾고, 그 뒤에 오는 것을 전부 `rootfs_data`라고 선언한다. 실제 파티션에
squashfs를 쓰는 보드라면 합리적인 규칙이다. 그런데 우리 squashfs 뒤에 있는
건 읽기 전용 loop 위의 256 KB 정렬 패딩이다. 결과는 이렇게 된다.

- `volume_find()`가 성공한다. 그래서 폴백이 발동하지 않는다
- 쓸 수 없는 몇 KB짜리 장치가 돌아온다
- `jffs2_reset()`이 거기에 `mkfs.ext4`를 시도한다

```
Filesystem too small for a journal
mkfs.ext4: I/O error while writing out and closing file system
factoryreset: /dev/loop2 will be erased on next mount
```

그리고 0을 반환한다. 아무것도 지우지 않고, 재부팅도 하지 않고. 모든 초기화
경로가 다시 조용히 죽었다. 1차 고장에서는 볼륨 레이어가 "모른다"고 답해서
폴백이 살 수 있었는데, 이번에는 모르면서 자신 있게 답하는 바람에 폴백이
무력화된 것이다.

최종 수정은 조건을 없애는 것이었다. 볼륨 레이어에 물어보는 것 자체를
그만두고, `/overlay`가 마운트되어 있으면 그 파일들을 지운다. 이건 마운트된
볼륨에 대해 `jffs2_reset()`이 하는 일과 정확히 같으므로 볼륨 기반 보드에는
영향이 없다. 덤으로 LuCI의 초기화 페이지도 고쳤는데, 종료 코드를 확인하지
않아서 위의 실패한 초기화를 사용자에게 성공으로 보고하고 있었다.

작은 후일담 하나. `firstboot -y`는 지우기만 하고 재부팅하지 않는다. 그런데
overlay를 지우면 ssh 호스트 키가 같이 사라지므로, 재부팅 없이 두면 다음
접속에서 사람이 당황한다. `-r`까지 붙여야 한다.

## 배운 것

복구 기능은 평소에 검증되지 않으니, 릴리스마다 실제로 눌러보는 수밖에
없다. 이 두 버그 모두 정상 사용 경로에서는 영원히 보이지 않았을 것이다.
그리고 "실패했는데 0을 반환한다"는 가장 나쁜 실패다. 초기화나 업그레이드처럼
결과를 눈으로 확인하기 어려운 기능일수록 종료 코드와 사후 상태를 함께
봐야 한다.

추상화 이야기는 따로 한 줄 적을 가치가 있다. 추상화는 모르는 걸 모른다고
말해줄 때만 안전망과 공존할 수 있다. `rootdisk`는 우리 배치를 이해하지
못하면서 자신 있게 답했고, 그 자신감이 폴백을 죽였다. 물론 원인의 절반은
loop-ext4-on-FAT이라는 우리 쪽 특이한 배치다. 추상화가 상정한 세계 밖에서
살기로 했으면, 추상화의 답을 의심하는 것도 우리 몫이다.
