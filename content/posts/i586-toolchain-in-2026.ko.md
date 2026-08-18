---
title: "2026년에 i586 툴체인 만들기"
date: 2026-08-17T10:00:00+09:00
draft: true
tags: ["openwrt", "toolchain", "gcc", "musl", "quark-x1000", "i586"]
summary: "Quark X1000은 i586에 MMX/SSE가 없고 x87 FPU는 있다. 이 조합을 위한 OpenWrt 툴체인의 첫 빌드는 컴파일러가 아니라 libc의 fabs()에서 멈췄다. 원인은 없는 명령어가 아니라 틀린 기본값이었고, 검증은 산출물 285개를 세는 것으로 끝냈다."
---

<!-- Part 2 of the LTE572W Yocto-to-OpenWrt porting series -->

## 타깃

Quark X1000의 코어는 Lakemont다. i586, **MMX/SSE 없음**, SMP 없음, 그리고
**x87 FPU는 있다.** 이 네 가지가 이 편의 결정을 전부 지배한다.

OpenWrt에 `x86/quark` 서브타겟을 새로 만들고 CPU 타입을 `lakemont`로
정의했다. 시작 플래그는 이랬다.

```
-march=lakemont -mtune=lakemont -Wa,-momit-lock-prefix=yes
```

`-momit-lock-prefix`는 단일 코어라 `lock` 프리픽스가 무의미해서 넣었다.

## 첫 빌드 실패

toolchain 빌드가 musl을 컴파일하다 멈췄다.

```
src/math/i386/fabs.c: error: impossible constraint in 'asm'
```

`fabs()`. libc에서 가장 단순한 축에 드는 수학 함수다. 왜일까. 명령어가
없어서? 아니다.

## 원인

GCC의 `-march=lakemont`는 **기본이 `-msoft-float`** 다. 컴파일러가 "이 CPU엔
FPU가 없다"고 가정하고 x87 레지스터를 아예 안 쓴다. 그런데 musl의 i386
포트는 수학 함수를 **x87 인라인 asm**으로 구현하고, 소프트플로트용 i386
변종이 없다. x87 레지스터를 요구하는 asm에 x87이 꺼진 컴파일러 — 그래서
`impossible constraint`다.

그리고 하드웨어에는 x87이 **실제로 있다.** 기존 Yocto 이미지가 i586 x87
유저스페이스를 잘 돌리고 있었고, 그 커널의 `MATH_EMULATION`도 꺼져 있었다.
오래된 ISA를 타깃할 때 문제는 "명령어가 없다"가 아니라 **툴체인의 기본값이
내 하드웨어와 다르게 가정한다**는 데서 온다.

## 수정

`-mhard-float`를 추가했다.

```
# include/target.mk — i386 블록
CPU_CFLAGS_lakemont = -march=lakemont -mtune=lakemont -mhard-float -Wa,-momit-lock-prefix=yes
```

플래그 하나에 다른 게 딸려 켜지지 않았는지는 `gcc -Q --help=target`으로
확인했다. **x87만 켜지고 MMX/SSE는 계속 꺼져 있어야 한다.**

한 가지 더 — soft-float에서 hard-float로 가는 건 부동소수점 **ABI 변경**이다.
`build_dir/{toolchain,target}-i386_lakemont*`와 staging을 지우고 처음부터
다시 빌드했다.

## 검증 — 믿지 말고 세어보기

빌드는 통과했다. 하지만 "플래그를 줬으니 됐겠지"는 검증이 아니다. 크로스
빌드에서는 어떤 패키지가 CFLAGS를 무시해도 빌드는 조용히 성공하고, 터지는
건 보드 위에서다. 그래서 산출물의 ELF 바이너리와 커널 모듈 **285개를
스캔**했다.

| 항목 | 결과 |
| --- | --- |
| MMX/SSE 명령 | 0개 |
| `lock` 프리픽스 | 0개 |
| libc의 x87 명령 | 있음 (= 하드플로트 정상) |

## 배운 것

- 오래된 타깃에서 싸움의 상대는 없는 명령어가 아니라 **틀린 기본값**이다.
  `-march=lakemont`가 soft-float를 기본으로 켤 줄은 켜보기 전엔 모른다.
- 컴파일러의 가정은 하드웨어로 반증할 수 있다. 잘 돌던 기존 펌웨어(x87
  유저스페이스, `MATH_EMULATION=n`)가 그 증거였다.
- 부동소수점 플래그는 ABI다. 바꿨으면 전부 다시 짓는다.
- 크로스 빌드의 검증은 플래그가 아니라 **산출물**에 한다. 285개를 세는
  시간이 보드에서 SIGILL을 쫓는 시간보다 훨씬 싸다.

이 단계에서 얻은 것은 부팅이 아니라 **정확한 바닥**이다. 모든 바이너리가 이
CPU에 실제로 있는 것만 쓴다는 확인. 다음 편은 그 바닥 위에서 커널이 안 뜨는
이야기다.
