# GPU / Display Rewrite Roadmap (hero2lte, iOS 16.7.16)

This document collects the remaining work on the Mali-T880 → DECON/VGR0 path.
It replaces the scattered local notes: everything here is in this repository.

## Hardware baseline

- **CPU:** 4× Exynos M1 (Mongoose) + 4× Cortex-A53 (ARMv8.0, no LSE, not
  I/O-coherent with GPU/DECON/UFS).
- **GPU:** Mali-T880 MP12, Midgard, 12 cores, 16×16 tiles, OpenGL ES 3.2.
  No native Metal.
- **Display:** DECON-F with hardware VGR0 scaler (750×1334 → 1440×2560),
  IDMA, s6e3hf4 panel.
- **Storage:** UFS 2.0 at 0x155A0000, 4 KiB sectors, up to 32 tags.
- **PMIC:** S2MPS16 on HSI2C15 (0x10550000).

## Current status and known problem

The phone boots to a usable Home screen with GPU, Wi-Fi, sound, touch and
physical keys. One cosmetic defect remains: flashing/rectangular "analog TV"
noise in the status-bar band, triggered when that band is repainted (Wi-Fi
spinner, Safari page load, scrolling, Home page 3+), at the lock screen and
before the Apple logo. The first two Home pages stay clean.

Evidence collected so far (all negative, so the work below starts from facts):

- CPU sampling of the IOSurfaces shows they contain data (seed stays 1).
- Clearing the D-cache on every bind (`shared_picture_fresh`) does not help.
- Forcing the CoreAnimation blur functions red colours the translucent
  backgrounds (dock, search, Safari bottom bar), not the noise.
- `glfetch=0`, `glzero_empty=1`, `herogpu_flush=0`, `glwait=0`,
  `herofb2=0`, zeroing FB2, `glskipblur=2` and iOS 18 were all tested.
- Android and LineageOS on the same panel are clean: the defect is in our
  software, not the hardware.

Gate before Level 1: a one-shot kernel snapshot of the 750×1334 source buffer
(`herosnap=N`, XNU patch `0388`) tells whether the noise is already in the
Mali-written frame or appears later in DECON/VGR0.

## Level 1 — GPU → Display (1–3 weeks)

### 1. Job batching and a kernel fence instead of `glFinish()`

`frame_end()` in `gpu/hakios-gl/src/eagl.c` currently calls `glFinish()`,
blocking the CPU ~4 ms/frame, and an animation produces ~1600 synchronous
submit ioctls/s (~0.5 ms each including syscall, lock and cache flush).

Midgard has three job slots (JS0 fragment, JS1 vertex/tiler, JS2 compute) with
HEAD/HEAD_NEXT/COMMAND_NEXT registers and a `next_job` pointer in each job
descriptor. Rewrite `herolte_gpu.c` to accept a batch of descriptors per frame
(in the style of `panfrost_ioctl_submit`), chain JS1 → JS0 inside the kernel,
and return a `fence_seqno`. Then:

1. Have `SweetFramebuffer.cpp` wait for that fence before the display flip
   (`PINGPONG_UPDATE` / VGR0 base address update).
2. **Only then** publish `HakiosWaitsForGPU = true` on the IOMobileFramebuffer
   registry entry: `display_waits()` in `eagl.c` then returns true and
   `wait_for_frames` goes to 0 without any boot argument.
3. Use `SHADER_PRESENT_LO = 0xff0f` only with a 5–10 ms power-off hysteresis;
   otherwise power-cycling 12 cores hundreds of times per second costs more
   than it saves on small UI quads.

Expected: ~4 ms/frame back to the CPU, one or two submits per frame.

### 2. Zero-copy swapchain on VGR0

`target_to_surface()` copies the whole 750×1334 frame (~4 MB, up to 240 MB/s,
1–16 ms spikes) into the display surface before VGR0 reads it again.

Allocate 2–3 physical buffers of ~4 MiB each (750×1334×4) and expose them to
`eagl.c` as the direct backing of backboardd's renderbuffers via SyncBO /
IOSurface. `frame_end()` then sends only fence + slot index to `SwapEnd`.

Partial damage is mandatory: CoreAnimation does not repaint every region each
frame (the `glsbcbar=1` black hole proved it). Either expose
`EGL_EXT_buffer_age`, force full-frame drawing (only ~1 M pixels on 12 cores
with VGR0), or blit the non-damaged tiles from buffer k-1 into k.

With VGR0 doing the scaling, 900×1600 or 1080×1920 sources become affordable.

### 3. Clean IOSurface transport

Today the seed is almost always 1, `WrapClientImage` (selector 6) is refused,
and `surface_changed()` samples one eighth of the rows per frame and skips
surfaces with planes > 0.

- In `IOSurfaceUnlock` (trap 3) increment the seed in the mapping shared with
  userland whenever `!(options & kIOSurfaceLockReadOnly)`.
- Implement selector 6 in `SweetSurface.cpp` on the model of selector 7, and
  always populate `planes[0]` (width, height, bytesPerRow, offset) so the
  `0 planes` case in `-texImageIOSurface:` disappears.
- Replace blind cache flushes with dirty-range maintenance (`dc cvac`, `dsb sy`).

## Level 2 — Exynos 8890 subsystems (3–8 weeks)

### 4. Heterogeneous scheduler, DVFS and TMU

- Model two clusters in `osfmk/kern/sched_*.c`: P = CPU 4..7 (Mongoose),
  E = CPU 0..3 (Cortex-A53).
- Keep backboardd, SpringBoard, mediaserverd (real-time audio), the UIKit /
  WebContent main thread and the touch thread on Mongoose; confine
  mobileassetd, installd, indexing and downloads to A53.
- Touch boost: raise the Mongoose frequency floor for 100–150 ms on the
  sec_ts/GPA1-0 interrupt.
- DVFS beyond vdd_mngs (BUCK2) and vdd_g3d (BUCK6): also govern MIF (LPDDR4 up
  to 1794 MHz, BUCK1/CMU_MIF) and INT (BUCK4/CMU_CCORE), because Mali and
  DECON both depend on memory bandwidth. Single lock on HSI2C15: raise voltage
  then `udelay(50-100 µs)` then PLL; on the way down reverse the order.
- TMU sensors for MNGS/APOLLO/G3D from the DTB, polled every 250 ms, with
  progressive step-down above 75–85 °C instead of abrupt throttling.

### 5. UFS 2.0 multi-queue

Use the 32 UTRD slots of UTRLDBR, interrupt-driven completion (IS.UTRCS on the
GIC) and a bitmask of in-flight tags so IOBlockStorageDriver can issue up to
32 concurrent requests. Build the PRDT directly from the physical pages of the
IOMemoryDescriptor (4 KiB sectors are already page-aligned): no bounce buffer.
This targets app launch, dyld_shared_cache reads and VM compressor paging.

### 6. Real display sleep

Stop SwapEnd/VGR0/GPU, send DCS `0x28` (display off) then `0x10` (sleep in)
with ~120 ms wait, put DSI into ULPS, clock-gate DECON/DSIM/G3D, switch off
LDO32/LDO33 (sec_ts), BUCK6 and S2MPB02 LDO16, then WFI/power-collapse the
clusters, waking on GPA0-5 (Home) / GPA2-4 (Power). Wake reverses the sequence:
`0x11` SLPOUT, re-init sec_ts, `0x29` display on, SwapEnd.

### 7. Recover physical RAM and tune the VM compressor

Audit `/reserved-memory` in the live DTB: return unused carveouts (CP shared
memory if the modem is off, camera ISP, secure media heaps) while never
touching el3_mon/TZ, 0xE2A00000 (S-Boot framebuffer), 0x91000000 (FB2/VGR0)
or ramoops. Target ~3.6–3.7 GiB usable instead of ~3.3 GiB. Tune WKdm/LZ4
thresholds so WebContent/SpringBoard large allocations do not stall
`vm_pageout`.

## Level 3 — Native Metal (2–4+ months)

Replace the legacy GLES bridge entirely: implement an MTLDevice /
MTLCommandQueue / MTLCommandBuffer / MTLRenderCommandEncoder /
MTLComputeCommandEncoder / MTLBuffer / MTLTexture shim backed by zero-copy
IOSurfaces/SyncBO, plus the QuartzCore SPIs (`[[color(0)]]`, presentDrawable:
into the VGR0 swapchain).

Compile Apple AIR (LLVM bitcode with `air.*` intrinsics) into Mesa NIR, then
use the existing `midgard_compile_shader_nir()` in
`mesa/src/panfrost/midgard/`. Midgard differs from Bifrost (VLIW vec4, rigid
sampler/UBO slots, no true bindless), so Metal 2 argument buffers need a
flattening pass in NIR. Framebuffer fetch maps to `LD_TILE` /
`nir_intrinsic_load_output`, which structurally removes the GLES
`gl_LastFragData` workarounds and opens the way to iOS 26 where the GLES path
no longer exists.

Start with a single CAMetalLayer triangle, then move QuartzCore to native Metal.

## Failed experiment: single-texture frame preservation (glpreserve)

A full-frame redraw was attempted by keeping one texture with the previous
frame and blitting it into the target at the start of every frame, so every
pixel would be written even where CoreAnimation does partial updates.
Result: **much worse** — most of the UI missing, flickering, chunks of screen
appearing and disappearing. Reason: the display uses **two rotating 750×1334
targets** (framebuffer 6 / texture 8 and framebuffer 7 / texture 11), while the
kept texture is single. At `frame_end` it saved buffer A, at `frame_begin` it
blitted that content into buffer B (and vice versa), so each buffer received
the *other* buffer's stale frame. The log confirmed increased texture/FBO churn
(`empty 0x1908` 122 vs 77, many more `glBindFramebuffer`) with no crash.

Lesson: preservation must be **per-buffer**, or the swapchain must render
genuinely full frames. Recycling one stale frame cannot fix a stale-content
bug — it propagates it. The code has been removed.

## Level 1.1 — implementation plan (kernel fence first)

Measured baseline (`boot-0382.txt`): ~1410 submits/s,
`GPU 2719 ms / submit calls 3209 ms` over 5 s, `glFinish()` ~4 ms/frame in
`eagl.c:frame_end()`. Mesa already has a userland async queue
(`HERO_ASYNC=1` in `gpu/mesa/herodrm.c:412-500`), so the caller no longer
blocks on each submit — but `glFinish()` still blocks the frame thread.

Order chosen to keep the boot stable:

1. **Kernel fence primitive (isolated, no behaviour change).**
   - `xnu/bsd/dev/herolte_gpu.c`: add `static uint64_t hg_seq;` incremented
     after every successful `hg_run()` in `herolte_gpu_submit()`; export
     `uint64_t herolte_gpu_fence(void)`.
   - `xnu/iokit/Kernel/arm/HeroGPU.cpp`: new selectors
     `kHeroGPUFence = 8` (returns the current seq) and
     `kHeroGPUWaitFence = 9` (input seq, waits under `hg_mtx` with a
     `pthread_cond`/`wait_result` until `hg_seq >= seq`; bounded, asleep).
   - `gpu/mesa/herodrm.c`: two thin wrappers around the new selectors;
     no caller yet.
   - Test: `hero_mtl_test`-style userland client reads the fence before and
     after a submit, checks it advances by one; `glframes=1` frame counters
     unchanged.

2. **Wire the fence into the display path.**
   - `gpu/hakios-gl/src/eagl.c:frame_end()`: when
     `HakiosWaitsForGPU` is published, skip `glFinish()` and hand the fence
     seq to `SwapEnd` instead.
   - `xnu/iokit/Kernel/arm/SweetFramebuffer.cpp`: before
     `PINGPONG_UPDATE` / VGR0 base-address change, wait for that seq.
   - Publish `HakiosWaitsForGPU = true` only after step 2 is working —
     otherwise DECON would read half-rendered frames (Gemini's ordering rule).
   - Measure: median frame time from `tools/compare_frame_times.py` must drop
     from ~10 ms without losing frames or breaking touch/Wi-Fi.

3. **Job batching.**
   - New selector `kHeroGPUSubmitBatch`: input array of `(jc, js)` pairs,
     output one fence seq; kernel takes `hg_mtx` once and programs several
     job chains in the Midgard slots.
   - Chain JS1 (tiler) → JS0 (fragment) through `JSn_HEAD_NEXT` /
     `JSn_COMMAND_NEXT` and the descriptor `next_job` pointer, so no userland
     round trip between the two stages.
   - `gpu/mesa/herodrm.c:submit()` emits one batch per frame instead of one
     call per job; keep the existing single-submit path as fallback.
   - Core affinity `SHADER_PRESENT_LO = 0xff0f` only with 5–10 ms power-off
     hysteresis.
   - Target: from ~1410 submits/s to 60–120 submits/s.

4. **Only then**: zero-copy VGR0 swapchain with dirty-rect preservation
   (Level 1.2) and clean IOSurface seed/wrap (Level 1.3).

Each step is a separate commit and a separate flash. Revert immediately if
frame time gets worse, frames tear, touch dies or Wi-Fi disconnects.

## Status (2026-11-28)

The current display source is **720×1280 @2x**, scaled exactly 2× to the
1440×2560 panel by VGR0 (`--canvas720` in `tools/mkadt_herolte.py`). Older
sections below describe the original 750×1334 baseline; the principles still
hold, but the numbers refer to that canvas.

### Done
- **Level 1.1 — fence**: `HakiosWaitsForGPU` published, `glFinish()` skipped
  (verified: `0392 wait_for_frames=0 sync_frames=1`). Median frame
  10.988 µs -> 7.081 µs, no tearing.
- **Level 1.2 — batch submit**: new IOKit selector 10, up to 8 jobs in strict
  FIFO, `hg_mtx` taken **per job** (the single-lock version caused 30.993 µs
  frames and choppiness — reverted). Mesa `HERO_ASYNC` worker drains the
  queue into batches, `HERO_BATCH=0` killswitch. Measured: 3328 submits /
  826 batch calls / 3312 jobs in 5 s, i.e. ~165 ioctl/s instead of ~1410/s,
  `submit calls - GPU` down from 490 ms to 92 ms per 5 s.
- **720×1280 @2x (720p)**: `--canvas720`, wallpaper class
  `360w-640h@2x~iphone`, root tune class, VGR0 stride derived from the width
  (0398), `mgcache.token` to force MobileGestalt, correct `autorun` (the real
  `hero-run`) in the drop zone. Faster than 750×1334 and an exact 2× scale.
- **UFS profiling (0399)**: `heroufspin` defaults to 80 µs; hundreds of
  commands with **0 sleeps and 0 ms asleep**. There is no sign of queue
  contention, so UFS multi-queue is **deferred** (Gemini strategy: low value).
  Frame median 11.646 -> 9.030 µs in 720p tests.
- **GPU clock**: `herogpu_mhz=419` (612.5 mV) is the accepted setting:
  0 `as_fault`, 0 soft resets, median frame 7.909 µs against 11.646 µs at the
  bootloader's 260 MHz. 650 MHz (700 mV) faults and soft-resets. 546 MHz
  (662.5 mV) has no faults and a better median (3.852 µs) but produced
  **immediate visual artifacts, lag and a wrong dock**, so it is rejected.
  The clock is set once at boot by `hg_clock_set()`; there is no runtime DVFS.
- **Display sleep (0400/0401)**: the conservative LDO-only path is committed
  but `herosleep=0` by default, i.e. off. True DSI SLPIN/DISPOFF plus sec_ts
  re-init is still open (the LDO-only attempt caused compressed snapshots
  and a UI restart).

### Not fixed
- The status-bar "analog noise" rectangles are not formally closed. They are
  less prominent with the 720p and cache fixes, but none of the targeted
  userland paths (see the evidence list above) removes them.
- Intermittent `backboardd` SIGSEGV with the same QuartzCore backtrace: 2
  crashes in **both** the 419 and the 546 logs, so the GPU clock is not the
  trigger. Symbols are redacted; not fixed.

### Remaining, in order of expected impact
1. **Zero-copy VGR0 swapchain (Level 1.2b)** — remove `target_to_surface()`:
   the full frame (~4 MB at 750×1334, less at 720×1280) is still copied into
   the display surface every frame and then read again by VGR0. Render
   straight into the 2–3 physical swapchain buffers.
2. **Per-buffer partial-damage preservation** — the `glsbcbar` black hole and
   the `glpreserve` failure prove CoreAnimation does not repaint the status
   band every frame. One kept texture was wrong with two rotating buffers;
   each buffer needs its own preserved copy (or genuinely full-frame draws).
3. **Clean IOSurface transport (Level 1.3)** — seed still 1 on unlock,
   `WrapClientImage` (sel 6) still refused, `surface_changed()` still skips
   planes > 0. Fix the seed in `SweetSurface::Unlock`, implement sel 6 with
   `planes[0]` populated, use dirty-range cache maintenance.
4. **Level 2** — cluster-aware HMP scheduler, MIF/INT DVFS and TMU, real
   display sleep, reserved-memory recovery. Performance and battery, not the
   visual defect.
5. **Runtime GPU DVFS** — `hg_power_thread` already does the CPU's
   mux/PLL/voltage switching and reads the GPU TMU. A GPU version needs idle
   gating (job slots plus `hg_mtx`) and a policy (419 base, boost under load,
   338/260 on screen-off or hot). Deferred: it would not fix the 546
   artifacts and a switch during a live job risks a bus hang.
6. **Level 3** — native Metal (AIR → NIR → Midgard). Months of work; removes
   the GLES bridge entirely and structurally avoids `gl_LastFragData`
   workarounds.

### Low value now
- UFS multi-queue: the 0399 profiling shows no sleeps and no queue contention.
- IRQ-driven wait in `hg_run` (Step 3 of the original batching plan): the
  batch already cut the overhead from 490 ms to 92 ms per 5 s, so the
  remaining time is real GPU work, not idle nap.

## Execution order

1. Keep the stable 720p / 419 MHz baseline; verify every flash with a log pull.
2. Level 1: zero-copy VGR0 swapchain with per-buffer damage preservation, then
   clean IOSurface seed/wrap. This is where the stale/noisy band most likely
   lives.
3. Level 2: HMP scheduler, MIF/INT DVFS + TMU, real display sleep, RAM
   recovery. UFS multi-queue only if profiling ever shows contention.
4. Runtime GPU DVFS, gated on GPU idle, as a separate reversible experiment.
5. Level 3: AIR → NIR → Midgard prototype.

Every step must keep the boot stable, Wi-Fi and touch working, and must be
measured against `tools/herolte-log-pull.py` frame counters before and after.
