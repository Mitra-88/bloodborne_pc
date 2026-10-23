# VRAM: where it goes, the warning, and what to optimize

Working notes for the Windows port, and the backlog for video-memory work. Facts marked
*(verified)* were read from the named code paths on 2026-10-23; sizes quoted from logs come from
the low-VRAM profile's measurements recorded in `run.py` (RTX 3050 Ti Laptop, 4 GB, Central
Yharnam), not re-measured for this document.

## 1. The budget the port actually uses

- `Instance::GetTotalMemoryBudget()` is the device-local heap total, fixed at startup
  (`vk_instance.h:506`). It is not the driver's live budget.
- Texture GC marks, computed once in `TextureCache::TextureCache` (`texture_cache.cpp:56-80`):
  - pressure = `max(min(VRAM - 0.6*min(VRAM, 8 GB), VRAM - 1 GB), 1.5 GB)`
  - critical = `max(min(VRAM - 0.2*min(VRAM, 8 GB), VRAM - 512 MB), 3 GB)`
  - below 8 GB the pressure mark is raised to 60 % of VRAM (`BB_GC_PRESSURE_PERCENT`, 10–75),
    because the game holds ~2.2 GB outside textures (backing, upscaler, swapchain).
  - trigger = `(VRAM - min(VRAM, 8 GB)) / 2`, which is **0 on ≤ 8 GB cards**: there
    `GarbageCollectImages` runs on every guest submission (hundreds per second).
- Overrides: `BB_GC_BUDGET_MB` (fixed budget on any GPU; `run.py` sets VRAM − 600 MiB for
  discrete cards ≤ 4608 MiB), `BB_LOW_VRAM=0/1`. On integrated GPUs the collector follows the
  driver's live budget instead (`GetDeviceMemoryBudgetNow`), because system-memory heaps dominate
  the usage there.

## 2. Where VRAM goes (verified)

1. **Guest memory backing** — `buffer_cache`. A 40-bit guest address space, 4 GiB sparse arena
   pages (min 1 GiB), sparse blocks ≥ 16 KiB. `EnsureResident` allocates one `vkDeviceMemory` per
   contiguous gap and binds it. Until 2026-10-23 nothing ever freed it, so it only grew; now
   `ReleaseIdleBacking` frees the ranges no binding has used for a minute (§5.1).
   - On NVIDIA the sparse buffer accepts VRAM memory types only, so a reserve of
     `min(512 MiB, heap/8)` is allocated at start when no system-memory fallback exists
     (`buffer_cache.cpp:96-127`).
   - On AMD/Intel the same code falls back to system memory when VRAM is full (slower, no crash).
2. **Textures** — one `VkImage` per image, all mip levels, uploaded on first bind. The `Memory:`
   line reports `textures N (M MiB)`.
3. **Outside both caches** — scene/render targets, FSR 3.1/4/4.1.1, DLSS, ML frame generation and
   the swapchain, at output resolution. Live resolution costs ~700 MiB more than the startup patch
   (`run.py:204`).
4. **Spill** — when an image cannot be allocated: retry without `WITHIN_BUDGET` (system memory),
   then with the other preferred heap, then fail (`image.cpp:189-214`). On failure the caller
   `TextureCache::ReclaimMemoryFor` (`texture_cache.cpp:1411`) frees clean, unbound, non-target
   images unused for 2 s and retries. AMD refuses system memory for render targets, so on 4 GB AMD
   cards this path is the last line of defence.

## 3. How eviction works today (so changes fit it)

- One `gc_tick` per guest submission; `TouchImage` stamps `lru_touched_tick` on every bind
  (`texture_cache.cpp:1370`), so "recently used" means "recently bound", not "visible".
- Not pressured: evict images unused for 20 s (`BB_GC_IDLE_SECONDS`), 10 per pass.
- Pressured: unused for 5 s (`BB_GC_PRESSURE_IDLE_SECONDS`), 20 per pass.
- Critical: unused for 1 s, then a final pass with 40 deletions and no age floor.
- Never evicted by age: GPU-written tiled images (their contents exist only in VRAM and cannot be
  written back); under pressure, clean render targets are `Touch`ed and kept. `BB_GC_DROP_OLD=1`
  drops minute-old ones and is **off** by default: dropping one caused a black screen with sound.
- On allocation failure, `ReclaimMemoryFor` evicts independently of the marks.
- Since 2026-10-23 the collector below the pressure mark runs at most 20 times a second instead of
  once per submission (`RunGarbageCollector`); above the mark every submission still sweeps. Ages
  are in `gc_tick`s, so the windows above are unchanged.

## 4. The warning (implemented, 2026-10-23)

An amber ImGui banner, ten seconds, top centre, no input capture, nothing blocked:
`BbOverlay::Warn` in `gpu/shim/bbport_overlay.cpp`, called from `GarbageCollectImages`.

| Knob | Default | Effect |
|---|---|---|
| `BB_VRAM_WARN_MB` | `1536` | free VRAM that triggers the warning; `0` disables |
| band | 256 MiB | warns again each 256 MiB closer to the limit |

- Limit = `BB_GC_BUDGET_MB` when set, else `GetTotalMemoryBudget()`.
- Re-arms only when free memory rises back above the threshold, so it cannot nag.
- Discrete GPUs only: on an iGPU the "limit" is system memory, not the VRAM crash.
- Logs `VRAM: N MiB free of M MiB, warned the player to save` on each warning.
- Known limits: `heapUsage` counts this process only, so other apps eating VRAM bring the crash
  earlier than the warning predicts; a warning pushed before the overlay is first initialised can
  expire unseen; no automated test (needs a GPU driven to its limit).

## 5. Backlog

Ordered by expected VRAM saved per unit of risk. Each item states the evidence, the seam to
change, the risks, and how to know it worked.

### 5.1 Free the guest-memory backing left behind by unloaded areas — **implemented 2026-10-23**

**Evidence.** It was the only allocation that never shrank, it grows as areas load, and on NVIDIA
it cannot leave VRAM at all. On a 4 GB card it ran out entirely before the reserve existed
("ErrorOutOfDeviceMemory" stopped the game).

**What the code does now** (`BufferCache::ReleaseIdleBacking`, `buffer_cache.cpp`):

- `EnsureResident` stamps `Backing::last_use` (the sweep epoch) on every resident range a binding
  asks for. `Backing::owned` says the range has its own `vk::DeviceMemory`, not a slice of the
  start-up reserve.
- One allocation per gap instead of one per call, so a range can be freed on its own.
- Once per `BB_ARENA_IDLE_SECONDS` (60; `0` restores the old never-free behaviour), **and only
  while usage is at or over the texture collector's pressure mark** (on a card with room the
  re-upload costs more than the space it frees), a sweep takes ranges that are `owned`, untouched
  for two sweeps (60–120 s) and **not GPU-modified** (both the tracker's readback bits and
  `gpu_modified_ranges`), at most 64 per sweep, then for each: adds
  `vk::SparseMemoryBind{memory = nullptr}`, zeroes its BDA page table entries
  (`runtime.FillBuffer`), marks the guest range CPU-modified so the next bind re-uploads it, and
  defers `vkFreeMemory` to the submission that carries the unbind (`scheduler.DeferOperation`, so
  the free cannot land before the GPU is done).
- GPU-modified ranges are skipped on purpose: writing them back needs a download, which submits,
  and the collector runs inside the submit callback.

**Known limits.** A range the GPU wrote (skinning output, compute) stays resident for the session.
Reserve slices (`owned == false`) are never freed, so the 512 MiB NVIDIA reserve is a floor. The
unbind has no prior-work wait (same as the bind path); 60 s of no bindings makes that moot.
Untested on hardware — see §7.

**Done check.** `Memory:` line: `game memory backing` falls after leaving an area, and
`GPU: released N MiB of game memory backing unused for 60 s (M ranges)` appears once a minute at
most; `Making range resident` does not repeat in a loop (a fault storm); texture uploads after a
release cost what the first visit cost.

### 5.2 Virtual texturing: per-mip residency — the actual "asset streaming"

**Evidence.** Images allocate and upload every mip level; only buffers use sparse residency today.
Large 4K textures are the bulk of `textures MiB`, and the game only samples high mips at close
range.

**Where.** `image.cpp` (`UniqueImage::Create/TryCreate`), `texture_cache` upload path,
`vk_instance` features, and a sampler-LOD/feedback source.

**How.** Create sparse-residency images, bind low mips at creation, bind high mips on demand.
Demand needs a signal the renderer does not have yet: either a GPU feedback pass (sample the
texture in a tiny pass, read back which mips were touched) or guest-side LOD knowledge. Without
feedback, unbacked mips sample as zero and the scene pops.

**Cheap fallback** (a tenth of the work, visible cost): for textures never used as render targets,
cap uploads to mip N and accept blur at distance behind a new quality knob: `BB_TEXTURE_MIP_CAP=N`.
**Not built** — the image is created with `info.resources.levels` and every view, upload loop and
size query assumes that count, so the cap has to be applied in all of them at once; there is no
GPU here to catch the Vulkan validation errors that a missed one produces.

**Done check.** Same scene, same output resolution: texture VRAM ≥ 20 % lower, no pop-in visible
during a 10-minute route at 1080p, and the fallback off by default.

### 5.3 Stop the every-submission GC walk on ≤ 8 GB cards — **implemented 2026-10-23**

**Evidence.** `trigger_gc_memory` is 0 there (§1), so `GarbageCollectImages` took `mutex` and
walked the LRU on every submission; the mutex is shared with guest fault handlers.

**What the code does now.** `RunGarbageCollector` skips sweeps closer than 50 ms apart while
under the pressure mark — 20 sweeps a second instead of hundreds — and sweeps every submission
once over it. `gc_tick` still advances on every submission, so the age windows in §3 are
unchanged. The "raise the trigger to ~10 % of VRAM" idea was dropped: a running scene is always
far above it, so it would not have skipped anything.

**Risk.** Up to 50 ms of extra latency before the collector reacts below the pressure mark; the
allocation-failure path (`ReclaimMemoryFor`) still covers a draw that cannot allocate.

**Done check.** Frame-time stats (`BB_FRAME_STATS=1`) unchanged within noise, pressure log
frequency unchanged when above a mark, `Memory:` still flat in a long session.

### 5.4 Prefetch and distance-based streaming — no data source, do not build

Textures are created when the game first binds a guest address; the game decides what exists. The
only camera signal in the port (`gpu/shim/bloodborne_cam.cpp`) writes yaw/pitch into the game's
camera and has no image→world mapping, so "load what the player is approaching" has nothing to
read. Proper prefetch needs hooks into the game's model/streaming manager (guest side), which is a
different project from the renderer.

### 5.5 Visibility-based eviction — already approximated, do not build yet

`lru_touched_tick` updates on every bind, and the 20 s / 5 s / 1 s windows already mean "recently
drawn". True on-screen visibility needs per-image world data (§5.4). The 4 GB log that motivated
this idea (≈30 re-uploads per second at 2.7 GB) predates the 60 % pressure rule and
`ReclaimMemoryFor`; re-measure before adding machinery.

### 5.6 Deliberately not worth it

- Dropping render targets on age (`BB_GC_DROP_OLD`): caused a black screen with sound; keep off.
- Per-mip hashing to avoid re-uploads: the contents must reach VRAM again after an eviction, so
  hashing would only save the hash.
- Making the NVIDIA reserve unconditional: on AMD/Intel it held ~480 MiB for the whole session
  unused ("reserve 0 of 478 MiB used").

## 6. Measuring

| What | How |
|---|---|
| VRAM by purpose | `Memory:` line every 60 s: total, textures, backing, reserve, idle, GPU-written tiled kept, process RAM |
| Every allocation by name/size | `BB_VMA_DUMP=<file>` writes allocator stats with each `Memory:` line |
| Frame times, stalls | `BB_FRAME_STATS=1` |
| Frame rate (kept vs. reverted) | `BB_FPS_LOG=1`: presents/s and the slowest frame, every 5 s — the number to compare before and after a tuning change |
| Simulate a smaller card | `BB_GC_BUDGET_MB=<MiB>`; `BB_LOW_VRAM=1` forces the low-VRAM profile |
| Simulate a full arena | `BB_ARENA_LIMIT_MB=<MiB>` |
| Eviction churn | `Texture cache: memory pressure, X of Y MiB (critical Z): N images evicted, M written back ... (every 5 s while under pressure)` |
| Backing released (only over the pressure mark) | `GPU: released N MiB of game memory backing unused for 60 s (M ranges)` |
| Backing release off / slower | `BB_ARENA_IDLE_SECONDS=0` (off) or a larger number of seconds |

### 6.1 Knobs, checked against the source (2026-10-23)

| Knob | Default | What it really does |
|---|---|---|
| `BB_GC_BUDGET_MB` | unset | Replaces the **total device usage** the marks compare against, and the VRAM warning's limit — not an image budget. `run.py` sets it only for discrete cards ≤ 4608 MiB. Setting it below current usage makes the collector evict and (since 5.1) release backing immediately; start at VRAM − 1.5 GB. |
| `BB_GC_PRESSURE_PERCENT` | 60, clamped 10–75 | Raises the pressure mark to this % of VRAM. **Only read when VRAM < 8 GB** (`texture_cache.cpp:72`), so on an 8 GB card it does nothing. |
| `BB_GC_IDLE_SECONDS` | 20, clamped 1–63 | Age at which an image goes while below the pressure mark. |
| `BB_GC_PRESSURE_IDLE_SECONDS` | 5, clamped 0–63 | Same, once over the pressure mark. |
| `BB_STAGING_KEEP_VRAM_MB` | 32 | **Blocks are 16 MiB** (`vk_staging_buffer_pool.cpp:29`), so the value is rounded down to a block count: 32 → 2 blocks, 16–31 → 1, below 16 → 0. Lowering it costs streaming bursts, not just headroom. |
| `BB_STAGING_KEEP_MB` | 512 | Host-visible upload ring (32 × 16 MiB), trimmed after 3000 idle frames: system RAM, not VRAM. |
| `BB_OBJECT_MOTION_CACHE` | off | Opt-in, read once. |
| `BB_SCENE_STENCIL_BITS` | auto | Forces the scene stencil format; a format change, not a saving. |

Two claims worth killing:

- **"Keep BC-compressed copies of textures"** — nothing is decompressed. Guest block-compressed
  formats map straight to `vk::Format::eBc*` (`liverpool_to_vk.cpp:718-750`) and the upload copies
  the guest's blocks, so textures already sit in VRAM in their compressed form. Further savings
  would need transcoding (BC7→BC1), which is a quality loss and GPU time.
- **"Fixed GB thresholds for the pressure and critical marks"** — they are derived from the device
  heap per GPU (§1); only the 1.5 GB / 3 GB floors and the `DEFAULT_*` values used when the
  instance cannot report memory are fixed. Because the comparison is against **live total usage**,
  bigger render targets and upscaler history already shrink what textures may use: resolution
  scaling of the budget is a no-op, not a missing feature.

## 7. Watch these when running 5.1 on hardware

5.1 shipped without a GPU to drive it, so the first sessions are the test. In order:

1. Does `game memory backing` fall after leaving an area, and does free VRAM rise with it? It only
   falls above the pressure mark; below it the release stays idle on purpose.
2. Does `Making range resident` repeat for the same range every few frames (a fault storm: the
   idle signal is wrong), and does the texture-upload count rise after each release?
3. Any visual corruption after re-entering an area — that would mean a range was freed whose
   contents the GPU had written, or the CPU-modified mark did not take.
4. `BB_ARENA_IDLE_SECONDS=0` must return the session to the pre-5.1 behaviour; use it to bisect.
