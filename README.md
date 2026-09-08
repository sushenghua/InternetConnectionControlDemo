# NetGuardian — Internet Connection Control Demo

> Daily internet-time control for the routers families already own.

A network-layer parental control system that enforces **per-child daily internet-time budgets** on the home router families already own. It needs no router replacement and no device-by-device enrollment, and the parent-facing UI never shows a MAC address.

It comes in two delivery modes:

1. **Web app**: a self-hosted parental-control console that drives the router directly from a server.
2. **ESP32 companion device + mobile app**: a low-cost board (the "Minion") plus a phone app, for households that own a router but no server.

Both modes share the same feature surface and the same vendor abstraction underneath.

## Features

- **Per-child daily time budget**: "2 hours/day on weekdays" holds across every device assigned to the kid.
- **Time-window scheduling**: combine multiple allowed windows per day (e.g. 15:30–18:00 + 19:00–20:30); outside the window is blocked regardless of remaining budget.
- **Multiple rules per group**: stack a stricter weekday rule and a relaxed weekend rule on the same child.
- **Network-layer enforcement**: blocks are pushed into the router's MAC filter list. Can't be bypassed by reinstalling an app, borrowing a sibling's device, or factory-resetting an iPad.
- **Auto-block / auto-reset**: counters freeze when blocked; at midnight everything resets and auto-blocks lift on their own.
- **LAN-wide device auto-discovery**: every client the router has ever seen, identified and labeled per child once.
- **Multi-vendor router abstraction**: a single `RouterClient` interface, vendor adapters underneath. UniFi + Asus working today; five more in development.
- **Manual override**: the parent can block/unblock any device at any time, distinct from the auto-block state.

<p align="center">
  <img src="docs/device/minion1.jpg" alt="The Minion — ESP32-C6 companion device" width="360" />
</p>
<p align="center"><em>The Minion — a USB-powered ESP32-C6 board.</em></p>

---

## Why this exists

School-age kids spend a lot of time online, and many find it hard to stop on their own. Most parents don't want to ban devices; they want a daily time budget that holds, configured once.

The existing options each fall short somewhere:

| Approach | Why it falls short for most families |
|---|---|
| **Built-in router parental control** (Xiaomi / Huawei / TP-Link admin pages) | Buried in admin UI; per-MAC; time-window only, no daily budget; UX differs by vendor. |
| **System-level limits** (iOS Screen Time / Family Link) | Must enroll every device. Cross-platform homes fragment. Easy to bypass on a borrowed device. |
| **Specialty network boxes** (Firewalla, Gryphon, Circle) | Expensive. Steep setup. Often requires replacing the router. |
| **Replace the home router** | Wi-Fi quality regression. Family-wide disruption. |

What's missing is a tool that is quick to set up, enforces a daily time budget at the network layer, and leaves the existing home router in place. NetGuardian tries to fill that gap in both delivery modes.

---

## Architecture

The two delivery modes share no backend; they are independent products with overlapping UX. The vendor abstraction (`RouterClient` interface + per-vendor adapters) has the same shape on both sides, so a new router type ports across with one adapter implementation.

### Mode A — Web app (server-deployed)

```
   Browser (parent)
        │
        │  HTTPS
        ▼
 ┌──────────────────────────────────────┐
 │  Next.js console  +  Node worker     │
 │  ├─ Auth, groups, rules, identities  │
 │  ├─ Cron poller (configurable)       │
 │  ├─ Policy engine (budget + windows) │
 │  └─ SQLite persistence               │
 └──────────────┬───────────────────────┘
                │  RouterClient  (vendor-agnostic)
                ▼
        ┌──────────────────┐
        │  Vendor adapter  │   UniFi · Asus · …
        └────────┬─────────┘
                 │  vendor admin API
                 ▼
          Home router  ──►  MAC filter / DHCP / association table
                 │
                 ▼
            Client devices
```

### Mode B — ESP32 companion + mobile app

```
   Mobile app  (iOS / Android, Next.js + Capacitor)
        │
        │  BLE  (pairing, telemetry, fallback when off-Wi-Fi)
        │  HTTP / WebSocket  (bulk data, control on Wi-Fi)
        ▼
 ┌──────────────────────────────────────────┐
 │  Minion — ESP32-C6  (ESP-IDF / FreeRTOS) │
 │  ├─ Wi-Fi + BLE provisioning             │
 │  ├─ CmdKey command interpreter           │
 │  ├─ Polling + accumulation engine        │
 │  ├─ Policy engine (budget + windows)     │
 │  └─ NVS-flash persistence                │
 └──────────────────┬───────────────────────┘
                    │  RouterClient  (same shape as Mode A)
                    ▼
          Home router  ──►  MAC filter
                    │
                    ▼
              Client devices
```

The mobile app and the firmware form a contracted pair: a single shared spec (`shared/protocol.md` in the coordinator workspace) governs BLE characteristics, HTTP endpoints, and message formats. Any wire-level change updates the contract first.

---

## Two delivery modes, same feature set

### Mode A — Web App (standalone, server-deployed)

A browser-based parent console. Runs on any always-on machine (mini-PC, NAS, home server) and talks directly to the router's admin API.

<table>
  <tr>
    <td align="center"><img src="docs/screen-shot/webapp-screen1.png" alt="Web app — Monitoring Groups with weekday/weekend rules, daily limits, and circular time-window dial." width="320" /></td>
    <td align="center"><img src="docs/screen-shot/webapp-screen2.png" alt="Web app — Identities list across discovered devices on the network." width="320" /></td>
  </tr>
  <tr>
    <td align="center"><em>Monitoring &amp; Rules</em></td>
    <td align="center"><em>Identities</em></td>
  </tr>
</table>

- Dark-mode console for setting groups, rules, and time windows.
- Two rules can stack per group (e.g. stricter weekdays, relaxed weekends).
- Identities tab consolidates every device the router has seen, with block/unblock per-MAC.

### Mode B — ESP32 Companion + Mobile App

For households that own a router but no server. The Minion is a thumb-sized ESP32-C6 board that joins the home Wi-Fi over Bluetooth pairing, then runs the same control loop on-device.

<table>
  <tr>
    <td align="center"><img src="docs/screen-shot/mobileapp-screen1.png" alt="Mobile app — iPad client showing 0m used / 120m left, with per-app data usage breakdown." width="320" /></td>
    <td align="center"><img src="docs/screen-shot/mobileapp-screen2.png" alt="Mobile app — Minion view with router link-up, Wi-Fi status, and 23 discovered clients with block controls." width="320" /></td>
  </tr>
  <tr>
    <td align="center"><em>Monitor (per-child usage)</em></td>
    <td align="center"><em>Minion (network &amp; clients)</em></td>
  </tr>
</table>

- **Plug in**: the Minion joins your Wi-Fi via Bluetooth pairing in the app. No re-cabling, no new SSID.
- **Auto-discover**: the Minion identifies every device on the network; label them to a kid once.
- **Set a daily budget**: e.g. "Up to 2 hours per day on weekdays." That's the whole rule.
- **Auto-block on overage**: when the budget is hit, the Minion pushes the kid's MACs into the router's filter list. The mobile app reflects state within ~2–12s. At midnight, the counter resets and the block lifts automatically.

---

## How it works

Both modes follow the same loop:

1. **Discover** every client on the LAN through the router's admin API.
2. **Poll** the router every ~60s and credit each tracked client's used-today counter.
3. **Enforce** when a child crosses the budget by pushing their MACs into the router's MAC filter list: no client-side software, so a reinstall can't bypass it.
4. **Reset** counters at midnight; auto-blocks lift on their own.

**Network-layer enforcement** is the design choice that distinguishes this from screen-time apps. Block lists live in the router's filter, so a borrowed device or fresh OS install doesn't bypass the rule.

### Supported routers

Drives the existing router via vendor admin APIs:

- UniFi
- Asus
- Xiaomi MiWiFi *(in development)*
- Huawei HiLink *(in development)*
- TP-Link XDR / AX *(in development)*
- OpenWrt *(in development)*
- Tenda *(in development)*

---

## Repository layout

This portfolio repo bundles three sibling projects via symlinks under `code/`:

```
code/
├── InternetConnectionControlWebApp/         # Mode A — standalone web app (server-deployed)
├── InternetConnectionControlEsp32Firmware/  # Mode B — ESP32-C6 firmware (ESP-IDF)
└── InternetConnectionControlMobile/         # Mode B — mobile companion app
```

> The three product repos may be kept local-only or moved to private GitHub repos. This portfolio repo exists to present screenshots, the device photo, and a product overview in one place.

---

## Hardware

- **MCU:** ESP32-C6 (Wi-Fi 6 + Bluetooth 5 LE, RISC-V)
- **Power:** USB-C, ~5 V

---

## Status

Working prototype across all three codebases. UI shown above is from the running web app and iOS app; the Minion photo is real hardware. This repo is the public landing page for the project; the three product codebases are kept private (available on request) and referenced here via symlinks.
