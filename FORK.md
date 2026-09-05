# M1-SOC-Armbian-Build — Fork Notes

This repository is a **fork of [armbian/build](https://github.com/armbian/build)**
(the upstream Armbian Linux build framework). The upstream README, build
instructions, and framework behavior are unchanged and still apply — see
[README.md](README.md).

This document explains **what this fork adds, why, and how to use it**. It is the
single place to look when you want to know "how does this differ from stock
Armbian?"

---

## What this fork is for

It adds first-class Armbian board support for the **Rockchip RK3308B-S EVB
M1SOC** — the SoC board inside the **Artillery M1 Pro "S1-SOC"** companion
display. That board is not supported by upstream Armbian, and mainline Linux
6.18 is missing several drivers/fixes it needs to boot with a working display,
touchscreen, thermal sensor, and Wi‑Fi.

The board is used as a **KlipperScreen + Crowsnest companion**: a small
480×272 touchscreen client that talks to a separate main Klipper/Moonraker
printer host. It does **not** run Klipper itself.

> **Role of this repo in the wider project.** This fork is the *source/compile*
> layer — it holds the board recipe, device tree, and kernel patches. The
> firmware-packaging scripts, overlays, and release tooling that *consume* this
> fork live in the separate private repo `M1-SOC-Armbian-Build-scripts-private`
> (and its sanitized public mirror `M1-SOC-Armbian-Build-scripts-public`).
> **Rule:** source code / DTB / driver changes go here; packaging scripts go in
> the scripts repo.

---

## Hardware summary

| | |
|---|---|
| SoC | Rockchip RK3308B-S (quad Cortex-A35, AArch64, ≤1104 MHz) |
| RAM | 512 MB DDR3 |
| Storage | eMMC (8-bit HS200) + microSD slot |
| Display | 4.3" 480×272 parallel RGB (DPI) panel |
| Touch | Goodix GT911 on I2C3 (`3-005d`) |
| Wi‑Fi | RTL8189CS (SDIO) |
| Audio | Internal RK3308 ACODEC |
| Console | UART3 (`ttyS3`) @ 1500000 baud |
| LEDs | GPIO0_A6 (power/green), GPIO0_A5 (heartbeat/blue) |

Full pinout/notes are in the board config header:
[`config/boards/rk3308bs-evb-m1soc.conf`](config/boards/rk3308bs-evb-m1soc.conf).

---

## What was changed vs upstream

### 1. New board recipe — `config/boards/`
- **`rk3308bs-evb-m1soc.conf`** — the Armbian board definition (board family
  `rockchip64`, kernel target `current`, U‑Boot `evb-rk3308_defconfig`, serial
  `ttyS3@1500000`, GPT layout, minimal/headless image, `BOOT_FDT_FILE` pointing
  at the kernel-compiled `rockchip/rk3308bs-evb-m1soc.dtb`).
- **`rk3308bs-evb-m1soc.dts`** — reference device tree for the board.

### 2. Kernel patches — `userpatches/kernel/rockchip64-current/`
These are applied by Armbian's kernel build (mirrored under
`userpatches/kernel/archive/rockchip64-6.18/` for the 6.18 archive branch). They
are the heart of the fork:

| Patch | Purpose |
|---|---|
| `0001-arm64-dts-rockchip-add-rk3308bs-evb-m1soc` | Adds the board DTS **and registers it in the kernel's Rockchip DTS Makefile** so Armbian compiles the DTB natively. |
| `0002-thermal-rockchip-rk3308bs-tsadc` | RK3308BS uses a **linear** TSADC conversion (kNum/bNum), not the rk3328 lookup table. Without it the SoC reads bogus-high temps and trips thermal-critical on a cold chip. |
| `0003-drm-rockchip-rk3308-vop-driver` | Adds the RK3308 **VOP** (display controller) register data/driver to the mainline Rockchip DRM driver. |
| `0004-panel-simple-simple-panel-compat` | `simple-panel` compatibility for the parallel RGB panel. |
| `0005-rk3308-rgb-platform-driver` | RK3308 **RGB/DPI output** platform driver glue. |
| `0006-panel-dpi-bus-format` | Sets the correct DPI bus format for the panel. |
| `0007-panel-dpi-probe-complete` | Completes panel-dpi probe for this panel. |
| `0008-panel-dpi-no-pm-runtime` / `0009-panel-enable-without-pm-runtime` | Make the panel enable without PM‑runtime (this board has no panel power-management runtime). |
| `0011-input-touchscreen-goodix-rk3308bs-reset-timing` | The GT911 needs a **much more generous reset timing** (500/100/500 ms) than upstream's tight T2/T3/T4; otherwise the chip never ACKs on I2C after reset. |
| `0012-drm-atomic-restore-panel-pre-enable-order` | Restores the pre‑enable‑before‑CRTC modeset ordering that a 6.18 upstream change broke — without it the VOP scans out to an unpowered panel bridge (board hard-hangs). |
| `0013-panel-simple-guard-optional-regulator` | Guards a NULL optional panel regulator in resume/suspend (paired with 0007, which made the supply optional) to avoid an Oops at first modeset. |

### 3. Disabled/trimmed upstream patch — `patch/kernel/archive/rockchip64-6.18/`
- `rk3308-vop-output.patch` was **trimmed to only its `rk3308.dtsi` hunk** (which
  other boards such as `sakurapi-rk3308b` depend on for `&vop` /
  `&display_subsystem` labels). Its `rockchip_vop_reg.c/.h` hunks were removed
  because they duplicate this fork's own `0003` VOP patch and caused a C
  redefinition build error. The original full patch is kept alongside as
  `.disabled-conflicts-with-rk3308bs-0003` for reference.
- See
  [`DISABLED-PATCHES-README.md`](patch/kernel/archive/rockchip64-6.18/DISABLED-PATCHES-README.md)
  for the full rationale and how to reconcile with upstream later.

> Everything else in this repo is stock upstream Armbian and is intended to stay
> that way, so the fork can be rebased onto newer upstream with minimal churn.

---

## How to build this board

Build exactly like stock Armbian, selecting this board. From the repo root:

```bash
./compile.sh default \
    BOARD=rk3308bs-evb-m1soc \
    BRANCH=current \
    RELEASE=bookworm \
    BUILD_MINIMAL=yes \
    KERNEL_CONFIGURE=no
```

The board recipe and the kernel patches above are picked up automatically —
`config/boards/rk3308bs-evb-m1soc.conf` selects them, and Armbian applies
`userpatches/kernel/rockchip64-current/*.patch` during the kernel build. The
output is a standard Armbian SD image plus the kernel/DTB artifacts.

> **Producing the final flashable companion firmware** (eMMC monolithic image,
> overlays, Klipper/KlipperScreen companion stack, release packaging) is done by
> the separate `M1-SOC-Armbian-Build-scripts-private` repo, which invokes the
> build above and then packs the result. If you only want the Armbian
> image/kernel/DTB for this board, the `compile.sh` command here is sufficient.

---

## Keeping the fork current with upstream

- Rebase/merge from `armbian/build` as usual. The board recipe (`config/boards/`)
  and kernel patches (`userpatches/kernel/`) are additive and rarely conflict.
- The only upstream file this fork modifies is the trimmed
  `rk3308-vop-output.patch`; re-check it after an upstream bump.
- If mainline gains RK3308BS support, the `0003`/VOP and panel patches should be
  diffed against upstream and dropped as they land, per the notes in
  `DISABLED-PATCHES-README.md`.
