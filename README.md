# Delta Portfolio — Systems & Engine Programming

![USPTO](https://img.shields.io/badge/USPTO-Patent%20Pending-005A9C?style=for-the-badge)

Application #19/641,687 — Method and System for Audiovisual Synchronization and Render Latency Minimization via Hardware Clock Domain Bridging

<br/>

**Notice: Core Source Code is Private**

This repository is a portfolio showcase.
The core source code for the custom D3D11 engine, kernel-level virtual audio driver,
AMD SVM hypervisor, and HFT trading bot is kept private for IP protection and security reasons.
*Full source code is available via ZIP submission upon request for job applications.*

<br/>

## Performance Metrics & System Logs

<img width="975" height="509" alt="image" src="https://github.com/user-attachments/assets/da00b32d-2b16-4574-a3df-d2f5b55873e9" />
<img width="3840" height="1080" alt="image" src="https://github.com/user-attachments/assets/e3373d5c-13de-4763-b3a8-f88d8c302a73" />
<img width="3840" height="1059" alt="image" src="https://github.com/user-attachments/assets/055ee063-5f73-4116-bc4c-cc370ec22fc0" />

<br/>

---

## Core Architecture & Features

<br/>

### Delta Engine — Custom C++ / D3D11 Game Engine

> A low-level multithreaded engine built from scratch to eliminate abstraction overhead, targeting nanosecond-precision judgment timing and frame-accurate rendering for a rhythm game.

**Architecture — 3-Thread Pipeline**

The entire loop runs across three threads with zero mutexes.

| Thread | Core | Role |
|---|---|---|
| Logic Thread | Core 1 | Fixed 16,000 Hz tick — input, judgment, scene update |
| Render Thread | Main | Waits on VBlank semaphore → D3D11 Present |
| VBlank Observer | Dedicated | D3DKMT kernel-level VBlank signal hooking |

- **Lock-Free Triple Buffering** : Three `RenderSnapshot` slots managed via `atomic<int>` swaps. Logic → Render data transfer with zero mutex overhead.

- **RDTSC-Based Nanosecond Timing** : `PrecisionClock` cross-calibrates `__rdtsc()` against QPC to measure CPU frequency precisely. All timing thereafter is nanosecond-resolution. `waitUntil` uses sleep above 2ms and `_mm_pause()` spin below.

- **SIMD-Parallel Judgment** : `NotePool` laid out as SoA (Structure of Arrays). `_mm256_cmp_pd` compares 4 notes per clock for miss detection. `[[likely]]` attributes guide branch prediction.

- **2GB Memory Arena** : 2GB pre-reserved via `VirtualAlloc` — zero heap allocation during gameplay. `NotePool` binds directly into the arena as a flat array.

- **Data-Oriented Scene Snapshot** : Scene writes state into `RenderSnapshot` each frame; the render thread reads snapshots only. Stale frames are automatically skipped via `scene_id` mismatch on scene transition.

- **Lua Skin System** : `.luaskin` files executed twice — first pass injects engine config, second pass does full parse. Load-time static OP culling and pointer pre-linking eliminate all runtime map lookups.

- **BGA Video Decoding** : Media Foundation runs on a dedicated decode thread, handing frames to the render thread via a lock-free triple buffer. The render thread calls only `uploadToGPU` for `Map/Unmap`.

<br/>

---

### Delta Cast — Kernel-Level Virtual ASIO Driver

> A virtual ASIO COM proxy driver that solves the broadcast capture problem caused by ASIO exclusive mode. Inserted between the game and hardware, it duplicates audio from the callback into a lock-free ring buffer and streams it to WASAPI Shared on a separate thread — with zero added latency on the primary path.

- **Zero-Latency Pass-Through** : The ASIO callback (`bufferSwitch`) calls only `ByteRingBuffer::Push` — no heap allocation on the hot path. No latency added to the game → hardware route.

- **Lock-Free Ring Buffer** : One 128KB `ByteRingBuffer` each for L and R channels. `atomic<size_t>` write/read indices enable mutex-free data exchange between the callback thread and the renderer thread.

- **Dynamic Limiter** : Release coefficient pre-computed as `exp(-1/(r*sr))` at setup. One gain calculation per frame prevents clipping at -0.1dBFS threshold.

- **Virtual Backend** : A software-clocked ASIO emulator requiring no real audio interface. Adaptive ±10µs clock correction based on ring buffer fill level (90% overrun / 10% underrun) prevents drift.

- **Proxy Backend** : Loads real ASIO hardware via `CoCreateInstance` and delegates all API calls transparently. Delta Cast self-registers under `HKLM\SOFTWARE\ASIO\Delta_Cast ASIO` via `DllRegisterServer`.

<br/>

---

### Delta HFT — Low-Latency Algorithmic Trading Bot (Binance Futures)

> A production-grade HFT bot applying the same low-latency systems design principles to financial markets. Real-time market regime classification drives dynamic routing between three structurally distinct strategy engines.

**Architecture — 4-Core Pinned Design**

| Core | Thread | Role |
|---|---|---|
| 0 | Config Watchdog | `inotify` file watch → `atomic<Config*>` pointer swap (Poor man's RCU) |
| 1 | Market Thread | Binance WebSocket `bookTicker` + `aggTrade` ingestion |
| 2 | Trade Thread | Order fill / cancel WebSocket ingestion |
| 3 | Engine Loop | Tick processing → regime detection → strategy engine → order management |

- **Config Hot-Swap (Runtime RCU)** : `inotify` detects `config.json` changes; `g_active_config.exchange(new_config)` replaces the entire config in a single atomic operation. Full parameter tuning without bot restart — no mutex on the hot path.

- **Market Regime Detection** : Every 60 seconds, volatility%, taker net delta, and OU (Ornstein-Uhlenbeck) β coefficient are combined to classify market state as CHOPPY / TRENDING / TOXIC, routing to a completely different engine per regime.

- **RollingZEngine (CHOPPY)** : Triple-filter mean reversion — rolling Z-score threshold + EMA OU coefficient gate + OFI (Order Flow Imbalance) direction confirmation. `RollingZScore` uses O(1) `sum`/`sum_sq` updates with periodic full recalculation to reset floating-point drift.

- **KinematicEngine (TRENDING)** : Price modeled as a physical system [position, velocity, acceleration]. `PhysicsState` updated via AVX2 `_mm256_fmadd_pd` Kalman update in a single 256-bit register. Jerk condition (rate of change of acceleration) ensures only accelerating trends trigger entry.

- **HawkesEngine (TOXIC)** : Self-exciting Hawkes Process models event clustering. `hawkes_energy` increments by `alpha` per event and decays as `exp(-beta*dt)`. Energy threshold breach triggers an OBI-directional entry to capture post-shock aftershocks.

- **Order State Machine** : 6-state FSM: `NONE → PENDING_ENTRY → LONG/SHORT → PENDING_EXIT → PENDING_EMERGENCY`. Ghost Fix (REST sync on WebSocket silence), two-tier stop loss (maker chase → market order fallback), and trailing stop built in.

- **Zero-Allocation Hot Path** : Order IDs generated directly into stack buffers via `std::to_chars`. Prefix matching uses 4-byte integer comparison (`0x5f746e65 == "ent_"`) instead of `strcmp`. Order response parsing uses the `simdjson` SIMD JSON parser.

- **Graceful Shutdown** : First SIGINT blocks new entries and attempts maker-order exit. Market order forced after 10s timeout. Second SIGINT triggers immediate termination.

<br/>

---

### Delta Visor — AMD SVM Type-1 Hypervisor (Kernel Driver)

> A kernel driver hypervisor built from scratch against the AMD SVM specification to verify hardware virtualization mechanics firsthand. Uses the "Blue Pill" technique — a single VMRUN instruction transitions the live Windows OS into a guest VM, with the hypervisor running transparently at Ring -1.

- **Blue Pill Virtualization** : `SetupGuestState` snapshots the full CPU state (CR/DR/GDT/IDT/MSR) into the VMCB at driver entry. VMCB.RIP is set to the `GuestResume` label and RSP to the current stack; after `VMRUN`, the OS resumes at `GuestResume` with no visible transition.

- **DPC-Based All-Core Simultaneous Virtualization** : `KeSetTargetProcessorDpc` + `KeInsertQueueDpc` dispatches DPCs to all logical cores simultaneously, virtualizing each core independently.

- **Pure x64 Assembly VM Loop (`Svm.asm`)** : Saves non-volatile registers → sets `MSR_VM_HSAVE_PA` → executes `VMRUN` → `clgi` (atomic interrupt gate) → calls `HandleVmExit` → loops back. VMMCALL with magic code `0xDEADBEEF` branches to `UnloadSequence` for clean teardown.

- **Full 4096-Byte VMCB Implementation** : Control Area (0x000–0x3FF) and State Save Area (0x400–0xFFF) per AMD APM Vol.2 Appendix B, implemented as a `#pragma pack(1)` struct. `static_assert(sizeof(VMCB) == 4096)` enforces spec compliance at compile time.

- **GDT Parsing & x64 Segment Handling** : `FillSegmentDescriptor` detects system segments (TSS/LDT) via the `S` bit and reads the additional 32-bit base from the 128-bit x64 GDT entry format. Granularity-bit-based limit expansion handled correctly.

- **GS Base Preservation** : `MSR_GS_BASE` (the Windows KPCR pointer) is backed up into R15 across the host loop and restored on unload, preventing kernel crash from a corrupted GS Base after VMEXIT.

<br/>

---

### Delta Tracker — Real-Time Hand Tracking ML Pipeline

> A from-scratch real-time hand tracking pipeline built on MediaPipe's TFLite models via the C API — no Python, no framework abstractions. Designed as the foundation for a commercial VTuber motion capture solution.


- **Custom SSD Anchor Decoding** : Palm detection anchor grid generated from scratch matching the model's [1, 2016, 18] regressor output. IoU + containment-based NMS removes duplicate detections post-inference.
- **Two-Stage Inference Pipeline** : Palm Detection (192×192 input) → ROI crop → Hand Landmark (224×224 input, 21 keypoints × 3 axes). Adaptive re-detection triggers on tracking loss, interval, or consecutive landmark failure — skips full detection when tracking is stable.
- **4-Stage Filter Chain** : Raw landmark output passes through: (1) 2D Kalman filter (position + velocity state, full 2×2 covariance in SoA layout) → (2) Post-Kalman gate (suppresses micro-jitter before EMA propagation) → (3) EMA low-pass → (4) Snap deadzone (sub-threshold deltas clamped to zero). All stages operate as SoA batch ops over all 21 keypoints × 3 axes simultaneously.
Lock-Free Triple-Buffered Architecture : Capture/Inference thread writes TrackingSnapshot via triple buffer atomic swap. Render thread reads the latest snapshot independently — inference speed never blocks display.
- **Zero-Allocation Inference Loop** : cv::Mat reused across frames (no per-frame allocation). Pre-allocated padded_buffer_ on the BinanceMarket-style stack avoids heap during JSON/landmark parsing. Reusable NMS vectors (det_buf_, nms_rects_) cleared without deallocation.
- **FramePool** : Lock-free camera frame pool decouples capture timing from render timing. Render thread always gets the latest frame without tearing.

<br/>

---

## Tech Stack

| Category | Stack |
|---|---|
| Language | C++20, x64 Assembly (MASM) |
| Graphics | Direct3D 11, DirectWrite, Direct2D |
| Audio | ASIO SDK, WASAPI (Exclusive / Shared), AVRT |
| System | Windows WDM Kernel Driver, AMD SVM (Hardware Virtualization) |
| Concurrency | Lock-Free Queue, Atomic, Triple Buffer, DPC, `std::jthread` |
| SIMD | AVX2 (`_mm256_fmadd_pd`, `_mm256_cmp_pd`, `_mm256_storeu_ps`) |
| Networking | Binance WebSocket, simdjson |
| Database | SQLite (WAL mode, Prepared Statement) |
| Platform | Windows 10/11 x64, Linux (Ubuntu) |
| ML / CV | TFLite C API, OpenCV, MediaPipe (Palm Detection, Hand Landmark) |

<br/>

---

## Repository Structure

```
Delta_Engine/     — Custom D3D11 game engine + BMS rhythm game
Delta_Cast/       — Virtual ASIO driver + WASAPI loopback renderer
delta_hft/        — AMD64 Linux HFT trading bot (Binance Futures)
Delta_Visor/      — AMD SVM Type-1 hypervisor kernel driver (.sys)
Delta_Tracker/    — Real-time hand tracking ML pipeline (TFLite + OpenCV)
```
