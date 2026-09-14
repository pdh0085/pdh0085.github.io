# 발행 순서 (Embedded Field Notes)

이 파일은 **연재 순서만** 담는다. 발행 여부는 여기 적지 않는다 — repo가 판정한다.
`main`의 `content/posts/<slug>.md`가 `draft: true`면 미발행, `draft: false`면 발행됨.

**다음 발행 = 아래 코드블록 순서에서 아직 `draft: true`인 첫 slug.**

이 방식이라 발행 후 이 파일을 고칠 필요가 없다. 순서를 바꾸고 싶으면 아래 목록만 편집하면 된다.

- 이 파일을 읽는 주체: `.github/workflows/scheduled-publish.yml` (GitHub Actions)
- 파서는 아래 코드블록 안에서 `^[a-z0-9][a-z0-9-]*$` 에 맞는 줄만 slug로 인식한다.
  `#`으로 시작하는 줄과 빈 줄은 무시된다. **코드블록 밖의 본문은 파서가 보지 않는다.**
- 진행 현황·발행 이력·편집 방침의 정본: 프로젝트 문서 `claude/blog-github-pages-status.md`
- 2026-09-14 이전에는 `publish-queue.local.md`(git 제외)를 로컬 Cowork 작업이 읽었다.
  발행이 GitHub Actions로 이관되면서 이 파일이 정본이 됐다.

## 순서

```
lte-port-forwarding-silent-mtu-failure
it-was-never-the-hardware
why-we-left-yocto
ab-rollback-drills
i586-toolchain-in-2026
one-usb-c-three-operating-systems
four-walls-to-first-boot
closing-the-brick-window
getting-the-rootfs-out-of-ram
zero-touch-in-101-seconds
boot-escalation-ladder
honest-router-numbers
silently-dead-recovery-paths
engineering-away-the-truck-roll
code-that-never-updates-in-the-field
four-grades-of-evidence
dissecting-a-190-second-boot
when-the-modem-says-connected
the-real-reason-rs232-was-dead
the-config-file-nobody-read
rebuilding-the-web-ui
hardware-is-the-truth
openwrt-webui-without-luci
chasing-320mhz-into-the-kernel
```

24편. 2026-09-14 기준 15편 발행 / 9편 대기.

`chasing-320mhz-into-the-kernel`은 LTE572W 연재가 아니라 MT7927 Wi-Fi 7 단발 기록이다.
사용자 결정으로 맨 마지막에 배치했다. 앞으로 당기려면 위 목록에서 위치만 옮기면 된다.

## 발행 페이스

월·수·금 08:00 KST (워크플로 cron `0 23 * * 0,2,4` — UTC 기준 일·화·목 23:00).
GitHub Actions의 예약 실행은 몇 분에서 십수 분 늦게 뜨는 일이 흔하다. 8시 정각이 아니라
8시 언저리에 나가는 것이 정상이다.

재고 9편이면 주 3회로 약 2026-10-02(금)에 소진된다.
`openwrt-webui-without-luci`부터 주 1회(월) 체제로 전환 예정 —
그때 워크플로 cron을 `0 23 * * 0` 으로 바꾸고, 안전망·모니터링 작업의 요일도 같이 바꿔야 한다.
