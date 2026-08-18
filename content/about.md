---
title: "About"
hidemeta: true
comments: false
showtoc: false
---

I'm a firmware engineer building cellular CPE products (5G/4G routers) on
OpenWrt and Yocto. This blog is where I write down real problems I hit in
the field and how I solved them.

Things I work on:

- OpenWrt / Yocto based router firmware (Qualcomm IPQ807x and others)
- Cellular modem integration: Quectel, Telit Cinterion, Sierra Wireless, u-blox
  and more — AT commands, QMI / MBIM, and USB networking modes (RNDIS / ECM / NCM)
- TR-069 / TR-369 remote management, GenieACS deployment
- Mass production test systems (Windows tooling, automated DUT control)
- Reliability architecture: watchdog ladders, A/B failsafe boot

## Working with me

I take on a limited number of consulting and contract projects alongside my
day job. Typical engagements are fixed-scope rather than hourly, so you know
the deliverable and the cost up front:

- **Remote debugging engagement** — you bring a misbehaving device or link
  (modem won't attach, throughput collapses, random reboots, "it works on the
  bench but not in the field"). We run 2–3 remote sessions on your hardware,
  and you get a written root-cause report with the fix or a concrete
  workaround. The kind of problems in these posts are exactly what this
  covers.
- **TR-069 / ACS deployment** — a working GenieACS-based remote management
  stack for your device fleet: provisioning flow, firmware upgrade path, and
  the operational runbook your team needs to keep it running.
- **Reliability / recovery architecture review** — a design review of your
  boot chain, watchdog strategy, and upgrade path, with a prioritized list of
  the failure modes that will actually generate support calls, and how to
  close them.

If your problem doesn't fit any of these boxes, write anyway — worst case I
tell you it's not a good fit and point you somewhere useful.

## Contact

- Email: **pdh0085@gmail.com** <!-- TODO: replace with your public contact address -->
- GitHub: [pdh0085](https://github.com/pdh0085)

A short description of the symptom and your hardware (SoC, modem model,
OpenWrt/Yocto version) is the fastest way to get a useful first reply.
