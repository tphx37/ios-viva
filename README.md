# ios-viva

**Apple's iOS 16 running on a Redmi Note 11 Pro 4G.**

ios-viva ports Apple's open-source XNU kernel (xnu-8796, the iOS 16 generation) to the Xiaomi Redmi Note 11 Pro 4G (`viva`, MediaTek Helio G96 / MT6781). An iOS 16.7.16 (20H392) userland derived from the iPhone X, with port-specific configuration and components, runs on top of it, from launchd to SpringBoard and the stock apps. It builds on [Nik300/ios-viva](https://github.com/Nik300/ios-viva), a Viva port of [ios-sweet](https://github.com/dipa-studio/ios-sweet) for the Redmi Note 10 Pro. The boot shim, patch series and tooling originate in those projects.

Graphics are hardware accelerated. Metal, and with it CoreAnimation, SpringBoard, Safari and the apps, runs on the phone's Mali-G57 MC2 through a separate driver project:

> **GPU driver → [Nik300/viva-gpu](https://github.com/Nik300/viva-gpu)**: a Metal driver for iOS 16 built on Mesa's Panfrost, with its own AIR → Valhall shader compiler.

> Experimental bring-up, not a usable phone OS. Apple's security model is mostly switched off, many services are stubbed, and installing it **erases Android's user data**.

## What works

| Area | State |
| --- | --- |
| Boot | Stock MediaTek LK → shim → XNU. Earlier Viva procedures used `boot_b`; check the current device state before any boot or flash |
| CPU | All 8 cores (6× Cortex-A55 + 2× Cortex-A76) via PSCI, with our own DVFS ([below](#dvfs-cpu-and-gpu-clocks-under-load)) |
| Memory | About 5.5 GiB, including the RAM between the SoC carve-outs |
| Storage | MediaTek UFS on LUN 2; the iOS root is an HFS+ volume in `userdata`, read/write |
| GPU | Mali-G57 MC2 kernel driver (`VivaGPUAccelerator`), Metal through [viva-gpu](https://github.com/Nik300/viva-gpu) |
| Display | 1080×2400 DSC panel, direct scanout, brightness and Night Shift. A recent snapshot measured ~60 Hz even with 120 Hz mode configured |
| Input | Goodix GT9896 touch, power and volume keys (MT6366 PMIC) |
| Power | Battery gauge, charger, Low Power Mode, RTC, panel sleep, shutdown and restart |
| Other | Wi-Fi (experimental, CONNSYS), hardware JPEG encoder, MDP scaler, USB link to a host (`tools/viva-link.py`) |

Audio output and haptics have been exercised on earlier builds (see [VIVA.md](VIVA.md)); their current state requires a fresh device check. Cellular, Bluetooth, camera, sensors, GPS and Face ID / SEP remain incomplete. There is no App Store or Apple ID.

The 2026-11-21 diagnostic snapshot found Wi-Fi connected but falling back to polling after an IRQ storm. Bluetooth and camera patches exist in source; their functionality on the currently booted kernel has not been established. Source commits, CI builds and device boot results are separate milestones; see [NOTES.md](NOTES.md) for the current handoff.

## DVFS: CPU and GPU clocks under load

When the port started, nothing touched the clocks. They stayed where the bootloader left them: A55s at 1866 MHz, **A76s at 1085 MHz** (slower than the little cores), CCI at 612 MHz and GPU at a fixed operating point. Stock Android does DVFS through the MCUPM firmware, which this port does not use. Instead the kernel runs its own governors on the application processor. They program the PMIC rails and PLLs directly, using the phone's own efuse-binned OPP tables from the live device tree.

### CPU governor (`osfmk/arm64/viva_cpufreq.c`, patches 0145, 0147, 0183, 0192, 0194)

![CPU DVFS control loop](docs/dvfs-cpu.svg)

- **Three domains**: LL (6× A55, 500–2000 MHz, Vproc12), big (2× A76, 774–2050 MHz, Vproc11) and the CCI (500–1400 MHz, kept at most two OPPs below big). Going up, the rail is raised and its readback awaited before the PLL changes. Going down, the PLL changes first. Vsram follows Vproc in hardware.
- **Load**: every 16 ms (100 ms while idle at the floor, with timer leeway so the wakeups coalesce), the governor reads the busiest CPU of each cluster. Two samples at ≥ 80 % jump to the ceiling. Otherwise it picks the slowest OPP that would be ≤ 50 % busy, and only steps down after 3 samples in a row.
- **Foreground vs background** (0194): the scheduler counts per CPU the time spent on low-priority threads (background and maintenance QoS). That time does not count toward the governor's decisions, and background work alone can raise a cluster to at most 1075 MHz. A single PowerLog SQLite thread had held the A55s at 2 GHz for hours.
- **UI boost** (0183, 0192): a UI thread that works 8 ms out of every 33 is only 25 % busy and never moved the clocks. Touch reports now boost both clusters for 250 ms, and presented frames give a lighter boost (1670 MHz) right after a touch or after a pause in drawing. While boosted, foreground threads prefer the A76s. Settings scrolling went from 42.9 to 19.9 ms per frame.
- **Ceilings**: LVTS die sensors (9 points, read every 256 ms) take one OPP off per sample at ≥ 75 °C and go to the floor at 105 °C. The battery temperature acts as a skin proxy and caps both clusters from 42 °C. Low Power Mode and display-off apply their own caps.

### GPU governor (`VivaGPU.cpp`, patches 0107, 0115, 0137)

![GPU DVFS control loop](docs/dvfs-gpu.svg)

- **49 OPPs from 300 to 950 MHz**, with VGPU and VSRAM_GPU raised before the MFGPLL. The rails move 50 mV at a time while jobs keep running (about 1 ms, gated). While the PLL relocks, the GPU runs from a 218 MHz fallback clock.
- **Load**: every 50 ms, the share of time a job chain was on the hardware. At ≥ 85 % it jumps to the fastest OPP. Otherwise it drops to the slowest OPP that would still be ≤ 70 % busy, but not within 200 ms of a step up.
- **Display boost** (0137): when the display has a frame queued that the GPU has not finished, the GPU goes straight to its fastest OPP. Bursty frames otherwise kept the average load, and with it the clock, low, and each frame of GPU latency costs a frame per second.
- **Ceilings**: `vivagpufreq` (floor), `vivagpufreqmax` and 550 MHz in Low Power Mode.

### Knobs

Everything is a boot-arg and can be changed live over the USB link (`cpu …`, `gpu …`), or from the Viva Manager app.

| Boot-arg | Default | Effect |
| --- | --- | --- |
| `vivacpufreq` | `1` | `0` leave the clocks alone, `1` governor, `2` manual (link only) |
| `vivacpull` / `vivacpubig` / `vivacpucci` | 2000 / 2050 / 1400 | ceilings in MHz |
| `vivacpuboost`, `vivacpuboostms` | fastest OPP, 250 | UI boost clock and duration (`0` ms = off) |
| `vivacpubg` | 1075 | ceiling for background-only load (`0` = off) |
| `vivacpufast` | `2` | A76 preference: `0` never, `1` always, `2` while boosted |
| `vivathermtrip`, `vivaskintrip` | 75, 42 | die and skin trip points in °C (`vivatherm=0` disables LVTS) |
| `vivagpudvfs` | `1` | GPU governor on/off |
| `vivagpufreq`, `vivagpufreqmax` | 300, 950 | GPU floor and ceiling in MHz |
| `vivagpuboost` | `1` | fastest GPU OPP while the display waits for a frame |
| `vivalpmcpu`, `vivalpmgpu` | 1530, 550 | big-cluster and GPU ceilings in Low Power Mode |

Over the link, `cpu` and `gpu dvfs` print the current OPPs, load, temperatures, boosts and time spent at each OPP.

## Building and booting

The engineering history and earlier procedures are in **[VIVA.md](VIVA.md)**; the current handoff is in **[NOTES.md](NOTES.md)**. The kernel is built by `.github/workflows/xnu.yml`; the iOS root is prepared separately from an iPhone X 16.7.16 IPSW. The GPU driver bundle is built and installed separately: [viva-gpu](https://github.com/Nik300/viva-gpu).

Historical flash commands in this repository are records of earlier experiments, not instructions to run blindly. Confirm the exact target, backups and each destructive operation with the device owner before writing any partition. Preserve bootloaders, recovery, modem, persist and backups. The current project instructions call for temporary boot images during bring-up. Do not push to the upstream `origin` without authorization.

Local Apple artifacts, device images and private logs live outside this repository in `../ios-assets/` and `../diagnostics-private/`; they are not distributed here. Keep them and device identifiers out of Git.

## Repository layout

| Path | What |
| --- | --- |
| `xnu/patches/` | XNU patch series, applied in order by CI. `0090`+ are viva's (MediaTek drivers, GPU, display, DVFS, Wi-Fi…) |
| `shim/` | Boot shim; `shim/viva.c` and `build-viva.sh` for this phone |
| `tools/` | ADT generator (`mkadt.py --viva`), boot image and root image builders, USB link host (`viva-link.py`) |
| `VIVA.md` | Device facts, build steps and the per-patch engineering log for viva |
| `NOTES.md` | Current Viva handoff followed by the historical ios-sweet log |

## Credits

Built on [ios-viva](https://github.com/Nik300/ios-viva) by Nik300 and [ios-sweet](https://github.com/dipa-studio/ios-sweet) by dipa-studio, Apple's open-source releases (xnu, IOHIDFamily, libpthread, hfs), [BearSSL](https://bearssl.org), [blacktop/darwin-xnu-build](https://github.com/blacktop/darwin-xnu-build) and [ipsw](https://github.com/blacktop/ipsw), and on Xiaomi's published `viva` Linux kernel sources for the hardware details. iOS itself is Apple's and is not distributed here.
