---
title: "첫 부팅까지 넘은 네 개의 벽"
date: 2026-08-17T10:00:00+09:00
draft: true
tags: ["openwrt", "quark-x1000", "kernel-config", "acpi", "serial-console", "debugging"]
summary: "커널은 잘 떴다. 콘솔이 안 보이고, eMMC가 없고, 이더넷이 없었을 뿐이다. Quark X1000 보드에서 OpenWrt 첫 부팅까지 넘은 네 개의 벽 이야기인데, 그중 세 증상은 꺼져 있던 커널 옵션 하나(CONFIG_ACPI)로 수렴했다."
---

<!-- Part 3 of the LTE572W Yocto-to-OpenWrt porting series -->

## 유저스페이스까지는 갔다

이 시리즈의 앞선 편에서 `x86/quark` 서브타겟과 툴체인을 만들었다. 이제 빌드된
커널을 GRUB 메뉴에 물리고 부팅할 차례다. 결과부터 말하면, 커널은 떴다.
`kmodloader: done loading kernel modules`까지 갔으니 유저스페이스 진입은
성공이다.

그런데 초기 부팅 출력은 깨진 문자였고, 어느 시점부터는 콘솔이 아예 침묵했고,
eMMC와 USB는 프로브에 실패했고, 이더넷 인터페이스는 존재하지도 않아서 PC가
IP를 받지 못했다. x86이라고 그냥 부팅되는 게 아니다. 그리고 이 정도는 임베디드
x86에서 이상한 일도 아니고, 오히려 정상적인 모습에 가깝다. 넘어야 했던 벽은 네
개였고, 뒤에서 보면 전부 같은 모양이었다. 커널이 펌웨어에게서 받아야 할 것을
못 받고 있었던 것이다.

## 벽 1: earlycon 출력이 깨진다

```
# 잘못된 earlycon (출력이 깨진다)
earlycon=uart8250,mmio32,$ADDR,115200n8
# 올바른 형태 — baud를 주지 않는다
earlycon=uart8250,mmio32,$ADDR
```

baud를 명시하면 커널이 표준 1.8432 MHz UART 클럭을 가정하고 분주비를 다시
프로그래밍한다. Quark의 UART 클럭은 그보다 훨씬 높아서 출력이 전부 깨진다.
baud를 빼면 GRUB이 이미 맞춰둔 설정을 그대로 물려받는다. 팁 하나.
`keep_bootcon`을 붙이면 진짜 콘솔을 고치는 동안에도 earlycon으로 전 과정을
볼 수 있다.

## 벽 2: 진짜 콘솔이 없다

`legacy console [ttyS1] enabled` 이후 무음. 답은 로그에 이미 있었다.

```
serial 0000:00:14.1: ignoring port, enable SERIAL_8250_LPSS to handle
```

Quark의 UART는 PCI 장치(8086:0936)이고, `CONFIG_SERIAL_8250_LPSS=y`가 있어야
잡힌다. 켜자 00:14.1이 ttyS0, 00:14.5가 ttyS1로 잡혔고 base_baud는
2764800(= 44.2368 MHz / 16)으로 보고됐다. 벽 1에서 baud를 주면 안 됐던 이유가
여기서 숫자로 확인된 셈이다.

## 벽 3: eMMC도 USB도 이더넷도 죽어 있다

증상은 세 갈래였다. sdhci가 `-16`으로 프로브 실패
(`genirq: Flags mismatch irq 0 ... (mmc0) vs. ... (timer)`), USB IRQ 경고,
그리고 이더넷 드라이버가 아예 프로브되지 않았다. 서로 무관해 보이는 세 증상을
따로 쫓다가, 로그 한 줄이 눈에 들어왔다.

```
APIC: ACPI MADT or MP tables are not detected
```

원인은 하나였다. **`CONFIG_ACPI`가 꺼져 있었다.** OpenWrt의 공유 x86 커널
config는 ACPI가 꺼져 있고, 모든 x86 서브타겟이 각자 다시 켠다. 새 서브타겟을
만들면서 그걸 놓친 것이다. 인과 사슬은 이렇다.

```
CONFIG_ACPI=n → MADT/_PRT 없음 → IOAPIC 미설정
             → PCI IRQ 라우팅 전멸 → sdhci(eMMC)·USB·이더넷 전부 실패
```

여러 증상이 동시에 나면, 증상 각각이 아니라 IRQ 라우팅처럼 밑바닥에 있는
공통 기반부터 의심해야 한다.

## 벽 3.5: ACPI를 켜도 부족하다

ACPI를 켜자 이번에는 `ACPI BIOS Error (bug): A valid RSDP was not found`.
Quark의 UEFI는 RSDP를 EFI 경로로 넘기는데, 이 보드의 GRUB 포크가 쓰는 커스텀
핸드오프는 그걸 메인라인 커널에 전달하지 않는다. 그래서 커널 파라미터로
주소를 강제했다: `acpi_rsdp=0x0f00e014`.

그 주소는 어디서 났느냐면, 정상 동작하던 Yocto 부팅의 `dmesg | grep RSDP`에서
읽었다. 이 시리즈에서 반복해 등장하는 기법의 첫 사례다. 정상 시스템이 옆에
있으면 그게 문서보다 정확한 참조 구현이다. 단, 이 주소 값은 이 펌웨어 빌드
기준이라 펌웨어가 바뀌면 달라진다. 그때마다 다시 읽어야 한다. 참고로
`pci=biosirq`는 통하지 않는다. UEFI 전용 보드라 레거시 PCI BIOS가 없기
때문이다.

## 벽 4: 그래도 이더넷이 없다

IRQ가 살아났는데도 eth0이 없다. Quark GbE는 PCI 8086:0937인데, 흔히 기대하는
`stmmac_pci.c`는 제네릭 Synopsys ID만 갖고 있어 이 장치를 모른다. 실제로 잡는
드라이버는 `dwmac-intel.c`, 즉 `CONFIG_DWMAC_INTEL`이다. 켜자 `intel-eth-pci`가
00:14.6을 eth0, 00:14.7을 eth1로 잡고 링크가 올라왔다.

최종 커널 커맨드라인을 요약하면 이렇다.

```
console=ttyS1,115200n8 earlycon=uart8250,mmio32,$ADDR acpi_rsdp=0x... reboot=p net.ifnames=0 biosdevname=0
```

## 보너스: 재부팅이 안 된다

위 커맨드라인의 `reboot=p`가 다섯 번째 이야기다. `reboot`을 치면
`reboot: machine restart` 직후 멈췄다. 이 커널은 `CONFIG_EFI=n`이라
`reboot=efi`는 빈 스텁이고, 그대로 `machine_real_restart(MRR_BIOS)`로 떨어져
레거시 리셋 벡터(0xFFFF0)로 far jump 하는데, 거기엔 BIOS가 없다. 게다가 그
함수는 `__noreturn`이라 뒤에 있는 CF9·트리플폴트 폴백이 아예 실행되지 않는다.
해결은 `reboot=p`, 즉 CF9 하드 리셋이다.

후일담이 하나 있다. 이 버그 때문에 `sysupgrade` 마지막의 `reboot -f`도 같은
경로를 타서, 이미지는 정상적으로 써졌는데 보드가 멈춰 "업데이트 실패"처럼
보였다. 필드 보드는 재부팅 전에 `echo pci > /sys/kernel/reboot/type`으로
구제할 수 있다.

## 배운 것

- 벤더(혹은 배포판)가 잘라둔 커널 config는 "이 하드웨어에 필요한 것"이
  아니라 "그 벤더가 필요했던 것"이다. 물려받은 config에서 꺼져 있는 옵션은
  전부 의심 대상으로 봐야 한다.
- 서로 무관해 보이는 증상이 여러 개면 공통 기반부터 확인한다. 이번엔 그게
  IRQ 라우팅이었다.
- 정상 동작하는 다른 펌웨어가 옆에 있다면 그건 문서보다 정확한 참조 구현이다.
  정답을 물어볼 수 있는 시스템이 있으면 거기서 읽어오면 된다.
