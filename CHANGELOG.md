# Changelog

## 1.0-hj43 — 2026-11-23 (selective merge of hj4/hj10/hj11/hj12/hj13 onto the hj3 `02` baseline)

Keeps the hj3 platform-entry and connector-patch adaptation from `03ff19d`, then brings over the later wake-relevant
changes that still make sense after hj7's DBUF write was disproved.

**Merged.**

* **hj4 AUX default:** `igfxhdmib-aux` defaults to `0`. The HDMI-stream AUX/DPCD block is still selectable with
  `igfxhdmib-aux=1`, but it no longer hides the EDID/link rebuild path Apple uses after wake.
* **hj10 software-only I2C counters:** `HDMIB-i2c` reports `enpw`, `gmbus`, `gpio`, `edidfull`, `pin` and `mode`.
  These are static counters only; they do not probe or write registers.
* **hj11 EDID cache:** a transient HPD low sample no longer drops the cached EDID before `orgHotplug()`. The sink is
  read again only when no EDID is cached; real monitor replacement is still covered by EDID content comparison and
  Apple's own DDC read.
* **hj12 DDC power-well removal:** `wrapGetDDCBlock()` no longer calls `ensurePW()` inside Apple's DDC transaction.
  The power-well write remains available for the own cold-boot EDID read path.
* **hj13 port-state preservation:** when `probePortState()` samples HPD low during a sleep/wake glitch, the previous
  `portState` is kept instead of forcing it to `0`. A genuinely unplugged sink is still retired by Apple's normal
  hotplug handler; `hpdlow` counts these low samples.

**Deliberately not merged.** The hj5/hj10-era DBUF slice-2 enable write is replaced by a read-only `sampleDbuf()`.
hj7 measured that after sleep Apple clears the slice-size bits of both slices, so copying slice 1 into slice 2 could
enable slice 2 with size 0 and leave both planes without a display buffer. The current build only samples
`REG_DBUF_CTL_S1/S2` into `dbufs1/dbufs2` for diagnosis.

**Diagnostic fields in this build.**

* `HDMIB-status`: `hotplugs`, `hpdlow`, `edid-ok`, `edid-fail`, `dbuf-calls`, `dc6-skipped`, `aux-blocked`,
  `cursor-blocked`, `fb`, `ddi`.
* `HDMIB-i2c`: `enpw`, `gmbus`, `gpio`, `edidfull`, `pin`, `mode`.
* `HDMIB-sentinel`: `fb`, `paths`, `flags`, `pwrwell2`, `portstate`, `dbufs1`, `dbufs2`.

No DPLL/pipe/PHY snapshot-restore, no reinject retry, and no forced complete modeset are added.

## 1.0-hj42 — 2026-11-23 (hj3 baseline restored for platform ID `0x8A5C0002` / `01005c8a`)

This branch starts from `03ff19d` (1.0-hj3) and removes the later hj38–hj41 sleep/wake register experiments.

**Why.** hj38 restarted from upstream rc6 and dropped the hj3 adaptation for the platform entry that sits behind
WhateverGreen's first-page platform-table scan. On `0x8A5C0002`, that made the connector lookup return null, so
the HDMI connector patch was skipped and the external display could disappear before sleep/wake repair code ever
ran. hj40/hj41 inherited that regression; the forced complete-modeset direction was therefore not a valid baseline.

**What is retained from hj3.**

* `HDMIB::injectDeviceProperties()` injects the `framebuffer-patch0-*` connector properties before
  `loadPatchesFromDevice()`.
* `build_weg_hdmib.sh` replaces `IGFX::findFramebufferId()` so the ICLLP platform list can be measured beyond one
  4 KiB page, reaching entry #20 (`0x8A5C0002`).
* The hj3 core hotplug, EDID, AUX/AGDC, cursor, VRR and DC6 hooks are unchanged.

**What is not in this build.** No DPLL/pipe/PHY snapshot-restore, no reinject retry, and no forced complete modeset.
Those wake experiments are kept out of this cold-start baseline until the `HDMIB-port` diagnosis again reports
`OK (HDMI)` and the monitor is recognized at boot.

## 1.0-hj3 — 2026-11-22 (v3: make WhateverGreen's own platform lookup reach the entry)

Built directly on the 1.0-hj1 baseline. One functional change plus a read-only diagnostic; nothing else
of v1 was touched. (1.0-hj2 was withdrawn: its private copy of the platform table was written by hand and
was never needed.)

**The root cause, measured on the binary.** WhateverGreen finds the entry for the requested
`ig-platform-id` with

```cpp
uint8_t *platformInformationAddress = findFramebufferId(framebufferId, gPlatformInformationList, PAGE_SIZE);
```

i.e. it only looks at the first `PAGE_SIZE` = 4096 bytes of the ig-platform-id table. On Darwin 25 the
ICLLP table is built at kext-init time by a static initialiser inside
`AppleIntelICLLPGraphicsFramebuffer` into its writable `__DATA,__bss` (`gPlatformInformationList` points
straight at the table, it is not a pointer to a pointer). The table holds 23 entries of `0x118` bytes
(`0x118` = `sizeof(FramebufferICLLP)`), the entry id is the first `uint32` of an entry:

| entry | id | offset | inside the 4096 byte window? |
|-------|--------|---------|------------------------------|
| #13 | `0x8A5C0001` | 3640 | yes → the connector patch was applied |
| #20 | `0x8A5C0002` | 5600 | **no** → the patch was never applied |

With `0x8A5C0002` the lookup returned `nullptr`, so WhateverGreen skipped its complete
`framebuffer-patch` block. The framebuffer kext therefore never received an HDMI connector: the
author's find/replace pair was never executed at all.

**The author's patch is correct for `0x8A5C0002`.** Entry #20 copies the very same connector table as
entry #12/#14/#19 (`__TEXT,__const` at `0x142668ac`, the shared three-port table that contains
`{3,10,1,1,0x400,0x2c1}`). It simply was never reached.

**What changed.** Only the size of the window: `IGFX::findFramebufferId()` now measures the table
instead of assuming it fits into one page. The table ends with a `0xFFFFFFFF` framebufferId — Apple
writes the terminator as `FFFFFFFF FFFFFFFF` on Darwin 25 (ICLLP) and as `FFFFFFFF 00000000`
elsewhere, and WhateverGreen's own `calculatePlatformListSize()` only knows the latter, which is why it
never finds the ICLLP one. Both are accepted; the last pair inside a bounded 8 KiB window is used, and
when no terminator is found the original one-page behaviour is kept.

Nothing in Apple's read-only memory is written: the entry and its connector rows live in the
framebuffer kext's writable `__bss`, so WhateverGreen's regular patch code writes to memory that is
meant to be written. The module itself does not touch the table at all — v2's private copy of it was
withdrawn.

**No risk of a wrong match.** The value `0x8A5C0002` does not occur in *any* data section of the
framebuffer kext — only as the immediate of the store that fills in entry #20 — so it can only ever be
found where it belongs. Verified by dumping every section of the extracted kext.

**Also in this build.** A read-only sentinel (`HDMIB-sentinel`: display path count, framebuffer online
flags, `PWR_WELL_CTL2`, `portState`). It re-reads offsets the module already uses and writes nothing;
it is there to show which state accumulates while the port loses its display path across sleep cycles.

## 1.0-hj1 — 2026-11-21 (huangjie fork, baseline "v1")
Upstream 1.0-rc6 with the two things the install guide asked the user to do moved into the driver. This is the baseline
every later change builds on.
* **No boot arguments.** The `-igfxhdmib` switch is gone: the module is active by default. Optional escape hatch
  `-igfxhdmiboff` disables it; nothing has to be added to `boot-args`.
* **No DeviceProperties.** The connector patch from the install guide
  (`framebuffer-patch0-find/-replace`, DDI 3/DP -> DDI 1/HDMI, GMBUS 2, type 0x800) is injected as virtual properties on
  the iGPU from `HDMIB::injectDeviceProperties()`, called at the top of `IGFX::processKernel()` right before
  `loadPatchesFromDevice()`. WhateverGreen's own patch path applies and validates it. If `framebuffer-patch0-find`
  already exists on the iGPU, the existing configuration wins and nothing is injected.
* Build changed from two to three edits to `kern_igfx.cpp`; `build_weg_hdmib.sh` validates all three.
* Built in GitHub Actions (`macos-15`), pinned to WhateverGreen `093e76f` + Lilu `0249c3a`.
* `HDMIB-version` reports `1.0-hj1`, so a stale kext is easy to spot.

## 1.0-rc6 ("v46") — 2026-11
* Colour-format fix is now **automatic, no boot-arg needed**: `igfxhdmib-rgb` defaults to -1 (auto). If the monitor's EDID advertises YCbCr (base byte 24 or CEA byte 3) the module hides it together with the CEA extension (the behaviour of rc5 `rgb=2`, confirmed on an LG 1366x768 monitor that was purple); monitors that do not advertise YCbCr keep their full EDID (CEA modes, audio flags).
* The decision is taken per monitor at each full EDID read, so swapping monitors re-evaluates it. `igfxhdmib-rgb=0|1|2` still forces a mode. `HDMIB-edid` shows the active mode (`rgb=`).
* First published build since rc3; rc4 and rc5 were development builds and never got their own release.
* The forced mode (`igfxhdmib-rgb=2`) is confirmed on hardware. The automatic decision is **not** yet: with the reference LG monitor and no boot-arg, `HDMIB-edid` should show `rgb=2` and `ext=0` and the picture should not be purple. If it is, set `igfxhdmib-rgb=2` explicitly and report the `feat=`/`cea3=` values.

## 1.0-rc5 ("v45") — 2026-11
* `igfxhdmib-rgb=2`: additionally hides the CEA extension block (count byte 0, checksum fixed) so the driver treats the sink as DVI/RGB, like the VGA adapter that worked. Driver log of the purple case: `Setting display mode ... encoded with 0x2` (HDMI AVI Y=2 = YCbCr 4:4:4) even after `rgb=1`.
* `HDMIB-edid` now also shows EDID feature byte, extension count, CEA byte 3 and the active rgb mode, so the installed build and the effect of the option can be checked.

## 1.0-rc4 ("v44") — 2026-11
* Fix: swapping the HDMI monitor kept the **previous monitor's EDID** (old mode list, wrong colour/purple tint). Causes: the cached EDID was kept whenever the quick revalidation read failed, and was never dropped on unplug.
  Now: unplug (HPD low) forgets the EDID; a failed or different revalidation triggers a full read; a sink that does not answer no longer keeps the old EDID alive; up to 6 DDC attempts per method.
* New `igfxhdmib-rgb` (default 1): hides the sink's YCbCr 4:4:4/4:2:2 support (EDID base byte 24, CEA ext byte 3, checksums fixed) so the driver keeps the link in RGB. Aimed at a purple/magenta tint seen with an LG 1366x768 monitor whose EDID advertises YCbCr. UNTESTED on hardware; `igfxhdmib-rgb=0` restores the old behaviour.
* New registry key `HDMIB-edid` (manufacturer, product, serial, year, blocks, colour-depth byte, count of forgotten EDIDs) to see which monitor the module believes is connected.

## 1.0-rc3 ("v43") — 2026-11 (hardening; the kernel-side changes were confirmed on hardware after the release)
* `isTarget` now also requires the HDMI connector-type bit (0x800): with `-igfxhdmib` but without the connector patch a real DP port on that DDI is no longer touched.
* `igfxhdmib-ddi` clamped to 0..5, `igfxhdmib-bootdetect` to 0..60 s; GMBUS pin from the platform table accepted for 1..12 (ICL type-C pins).
* New registry keys: `HDMIB-version`, `HDMIB-port` (what the driver really has on the target DDI, with an explicit "NOT HDMI" diagnosis).
* New `hdmib_setup.sh` (macOS only): `check`, `patch <config.plist>` (backup + DeviceProperties + boot-arg), `verify <kext>`. No Linux needed for the 01005c8a preset.

## 1.0-rc2 ("v42") — 2026-11
* macOS 15 Sequoia (Darwin 24) layout marked **verified** (static check + HDMI run on the development machine, 15.8.2 / 24H232).
* `tools/layout_fingerprint.py diff` now judges each `Layout` offset by whether the same functions still use it (a moved field changes that set) and reports
  changed code in hooked functions as information, instead of failing on any difference. Added `reference_darwin24.json`, `verify_layout.sh`.

## 1.0-rc1 ("v41") — 2026-11
Clean-up and generalisation of the development builds v1–v40. Behaviour on the development machine is intended to be identical to v40's
`igfxhdmib-mode=16` configuration (same hooks, same order of operations); everything that only existed to investigate the driver was removed.

* Removed: ~700 lines of research instrumentation (register samplers, ring/PS-counter tracing, AGDC/mode-enumeration observers, DDC self-test,
  RC6/CPG experiment) and the per-mode (`igfxhdmib-mode`) branching — mode 16 behaviour is now the only behaviour.
* Removed: the hard-coded EDID of the development monitor. If the monitor's EDID cannot be read the stack now gets Apple's own answer; an optional generic
  1024x768@60 EDID is available with `igfxhdmib-edid-fallback=1`.
* New: one `Layout` table with every private-structure offset, keyed by Darwin major version. Unknown or unverified versions leave the module idle instead
  of risking a kernel panic (`igfxhdmib-force=1` overrides).
* New: target DDI / framebuffer index / GMBUS pin are no longer fixed to "DDI B, FB2, pin 2": `igfxhdmib-ddi`, `igfxhdmib-fb`, and the pin is read from the platform table.
* New: only the Ice Lake LP framebuffer kext is touched; other GPUs are unaffected even with `-igfxhdmib` set.
* New: required symbols are checked before any hook is installed; essential and optional hooks are routed separately.
* New: `HDMIB-layout`, `HDMIB-hooks`, `HDMIB-status` in the registry for bug reports.
* New: `tools/layout_fingerprint.py` to validate the layout table against another macOS build.
* Defaults now equal the development machine's tested configuration: `bootdetect=3`, `hwcursor=1`.
* Fix: a second EDID block is only returned when it was actually read (v40 could return an all-zero block).

## 0.x ("v1–v40")
Research builds: HPD-only detection, AGDC plumbing, EDID reader (GMBUS/GPIO), DBUF slice 2 power-up, hardware cursor for the HDMI framebuffer, boot-time
detection, AUX short-circuit, DC6 panic fix, VRR off, plus instrumentation. Not published.
