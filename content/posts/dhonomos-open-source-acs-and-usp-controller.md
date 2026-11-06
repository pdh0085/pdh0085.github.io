---
title: "Dhonomos: An Open-Source TR-069 ACS and USP Controller in One Rust Binary"
date: 2026-11-06T19:00:00+09:00
draft: false
tags: [tr-069, tr-369, rust, open-source]
categories: [remote-management]
summary: "I have released Dhonomos, a TR-069 (CWMP) ACS and a TR-369 (USP) Controller in one Rust program, under Apache-2.0. Why it exists, what it does today, and what it does not do yet."
---

If you work on home routers, LTE/5G CPEs or ISP operations, you know the two standards behind
remote device management: **TR-069** (CWMP), the SOAP-over-HTTP protocol that most CPEs in
the field speak, and **TR-369** (USP), its successor, which runs over MQTT, WebSocket or
STOMP. The server side of TR-069 is called an ACS; the server side of USP is a Controller.

I have just published **Dhonomos**, an ACS and a USP Controller in one Rust program,
under the Apache-2.0 license:

**<https://github.com/pdh0085/dhonomos>**

![The device list of the Dhonomos web UI, showing simulated devices](/images/dhonomos/devices.png)

This post explains why it exists, what it does today, and what it does not do yet.

## Why Another ACS?

I am a software developer at Lightspeed International, a Taiwanese maker of routers and
LTE/5G CPEs. Testing firmware means talking to the devices the
way an operator's server does: reading parameters, changing settings, pushing firmware, and
waking devices that sit behind NAT. For that we needed a server we could run in the lab, read
the code of, and change when a device did something unusual.

The open-source choices were few. Commercial ACS products are expensive and closed. The
best-known open-source ACS is licensed under the AGPL, which many companies will not run
or modify. And open-source USP Controllers that a small team can actually deploy are rare,
even though USP Agents (such as the open-source obuspa) are already in products.

So Dhonomos started as a test helper in our lab and grew into a full server. The company
agreed to release it as open source.

## What It Does

**CWMP ACS**

- TR-069 Amendment 6 sessions (CWMP 1.0-1.4) over TLS, with Digest or Basic auth.
- Per-device credentials bound to their device, optionally issued on first contact. After
  that, the shared credential no longer works for that device.
- Zero-touch onboarding: a new device is read in full on its first Inform, under a policy
  (accept all, quarantine, pre-registered only, OUI allow-list).
- Connection Requests over HTTP, over UDP with a built-in STUN server, and over **XMPP**
  with a built-in XMPP server. XMPP matters for CPEs behind carrier-grade NAT: the device
  keeps a TCP connection open to the server, and the wake-up goes down that connection with
  a delivery confirmation.
- A built-in file server for firmware and configuration files, with signed links and
  bandwidth limits.

**USP Controller**

- MTPs: MQTT 5 (through an external broker such as Mosquitto, or a broker built into
  Dhonomos), WebSocket in both directions, and STOMP 1.2 with a built-in broker.
- Its own CA for Agent certificates, with issuing, an expiry list and revocation. An
  EndpointID that has a certificate cannot be claimed over a weaker MTP.

**Operations**

- A task queue with retries, presets (YAML or web UI) and sandboxed provisioning scripts in
  [Rune](https://rune-rs.github.io/), a small embeddable language for Rust.
- Bulk jobs and firmware campaigns with canary stages and an automatic pause when too many
  devices fail.
- Diagnostics with one result format for CWMP and USP: ping, traceroute, DNS, TR-143
  speed tests, Wi-Fi neighbours.
- Configuration backup, drift detection, restore and device replacement.
- WireGuard tunnels to devices, managed from the server, with an optional hub program.
- A REST API with an OpenAPI 3.1 document, a web UI (SvelteKit) built into the binary,
  roles, device scopes per team, OpenID Connect single sign-on, TOTP two-factor
  authentication and an audit log.

The whole server is one executable plus PostgreSQL. The container image is a static musl
binary on an empty base image, about 22 MB.

## How It Performs

On a 12-core Xeon E5-2630 (2.3 GHz) with PostgreSQL on the same host, a simulator drove
**50,000 virtual CPEs at about 2,000 periodic Informs per second** for 60 seconds: no errors,
Inform p50 9.9 ms and p99 441 ms, and **113 MB** of memory for the server. That run used a
larger database pool and session limit than the defaults (96 connections, 4,096 concurrent
sessions); with the defaults the same load gets 5.7% of sessions turned away with 503. The
simulator and the benchmark harness are in the repository (`crates/simulator`,
`cargo xtask bench`), so you can repeat the measurement.

## Why Rust

A management server sits on the internet edge of an ISP and parses input from tens of
thousands of devices it does not control. Memory safety matters there, and so does a small,
predictable footprint. Rust gave us both, plus a strong ecosystem for the parts we needed:
tokio and axum for the network side, sqlx with compile-time checked queries for PostgreSQL,
prost for USP protobuf, rustls for TLS. `unsafe` is forbidden in every crate but one,
which has a single, documented `getsockopt` call, and the protocol parsers have fuzz targets.

## What It Does Not Do Yet

Honesty first:

- It is **pre-1.0**. There are no releases yet; you build it from source. Flags, the API and
  the database schema can still change.
- There has been **no external security review** yet, only internal ones.
- The operations guides are in Korean; the README and the installation guide are in English.
- Full multi-tenancy is not there yet.
- It has been tested with OpenWrt-based CPEs and with simulators. Devices from other vendors
  may show behaviour we have not met yet. Reports are very welcome.

## Try It

```bash
git clone https://github.com/pdh0085/dhonomos.git && cd dhonomos
docker build -t dhonomos:dev .
cp deploy/dhonomos.env.example deploy/dhonomos.env   # fill in the secrets, CPE credentials, ACS cert
docker compose -f deploy/docker-compose.yml up -d
```

Then point a CPE's ACS URL at the server. The
[installation guide](https://github.com/pdh0085/dhonomos/blob/main/docs/guide/install.en.md)
walks through TLS, the first CPE and the first USP Agent.

## The Name

Dhonomos is "DH" (my initials) plus *oikonomos*, the Greek word for the steward of a
household, and the root of the word *economy*. A server that looks after the devices in
subscribers' homes seemed a fitting job for a steward.

If you run an ACS today, build CPEs, or are looking for a USP Controller, I would like to hear
what you need. Issues are open on GitHub.
