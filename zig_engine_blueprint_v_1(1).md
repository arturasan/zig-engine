# Zig Engine Blueprint (Full, Zig‑Only) — v1.12

## 0) v1.12 Amendments (Normative)

This section **supersedes** earlier sketches where they conflict. It locks down ABI details, hot‑reload contracts, and a few platform specifics.

### 0.1 Single exported entry point for the game module

Export exactly **one** C symbol from the game module that returns the `GameAPI` function table:

```zig
pub const GameABI_Revision: u32 = 1;

pub export fn game_get_api(abi_revision: u32) callconv(.C) ?*const GameAPI {
    if (abi_revision != GameABI_Revision) return null;
    return &GAME_API; // `GAME_API` is a static const GameAPI filled by the game.
}
```

Host loader must retrieve and validate it **before** any calls:

```zig
const std = @import("std");
const api = @import("../engine/api.zig");
const GetApiFn = *const fn(u32) callconv(.C) ?*const api.GameAPI;

// NOTE: `path` is the game module path (e.g., the result of resolveGameModulePath())
var lib = try std.DynLib.open(path);
const get_api = lib.lookup(GetApiFn, "game_get_api") orelse return error.MissingSymbol;

const g = get_api(api.GameABI_Revision) orelse return error.ABIMismatch;
if (g.abi_revision != api.GameABI_Revision) return error.ABIMismatch;
if (g.abi_magic != 0x47414D45) return error.BadMagic; // 'GAME'
if (g.size != @sizeOf(api.GameAPI)) return error.BadSize;
if (g.version == 0) return error.BadVersion;
```

> Note: Keep `lib` as a field of the hot-reloader and **close it on unload**; do not let it go out of scope while the game API is in use.

### 0.2 Text encoding & lifetimes across the ABI

- **All strings/text are UTF‑8.**
- Unless otherwise stated, `ReadOnlySpanU8` text is **valid for the current frame only**. Copy if you need it later.
- Never pass Zig slices/allocators across the ABI.

### 0.3 Revised `EngineServices` (asset lifetime + job scheduling)

Add explicit asset retain/release and tag job ownership so quiesce can target only game work.

```zig
pub const JOB_OWNER_ENGINE: u32 = 0;
pub const JOB_OWNER_GAME:   u32 = 1;

// use api.JobDesc (defined in src/engine/api.zig) to avoid type drift here

pub const EngineServices = api.EngineServices; // canonical definition lives in src/engine/api.zig (§5.3)
```

**Thread-safety contract (normative):**

- `log` and `schedule_job`: callable from worker threads.
- `asset_acquire`/`asset_release`: thread-safe.
- `asset_get_by_guid`: thread-safe; may block on IO.
- `ecs_query_*`: **main thread only** unless stated otherwise.

**Asset handle rules (ABI):**

- `asset_get_by_guid` returns an **asset ID handle** (`AssetHandle`) on success; the caller **must** eventually call `asset_release(handle)`.
- `asset_acquire`/`asset_release` operate on **handles** and are idempotent in Debug with asserts for underflow; host refuses reload until all game‑owned references reach zero.
- Handles are opaque; do not assume pointer stability; **never** cache internal pointers across reloads.

**Quiesce (reload) targeting:**

- The host stops accepting new jobs with `owner_tag == JOB_OWNER_GAME` and waits only on those. Engine/editor jobs continue.

### 0.4 Temp allocator semantics (cross‑ABI)

The `temp_alloc/ temp_free` pair is a **frame‑scratch** facility:

- Intended for **short‑lived, frame‑local** buffers crossing the ABI.
- **Not thread‑safe.** If used from worker threads, pass ownership back to the main thread or free within the same worker before frame end.
- The host may cap single allocations (e.g., 256 KiB); returns `null` when over limit. Always check for `null`.
- Every successful `temp_alloc` **must** be paired with `temp_free` **within the same frame**.

Rename in code if helpful: `frame_scratch_alloc` / `frame_scratch_free` (ABI signature unchanged).

### 0.5 Event Bus clarifications

- Provide generated **pack/unpack helpers** per event type and static assertions on header `{ size, align }`.
- **Unsubscribe requirement:** The game **must** unsubscribe all callbacks during `on_unload`. The host asserts this in Debug.
- Suggested header:

```zig
pub const EventHeader = extern struct { kind: u32, size: u32, align: u32, _reserved: u32 };
```

- `EventHeader.align` is a **byte** alignment (power-of-two) and must be ≤ 64; add compile-time asserts in pack/unpack helpers.

### 0.6 ECS/Scene across reloads

- `*Scene` is owned by the host and remains valid across reloads.
- The game **must not** cache raw component pointers across reload; use queries/iterators each frame or store **handles/IDs** only.

### 0.7 `GameResult.recoverable` policy

- `ok` → continue.
- `recoverable` → host logs the error, **pauses the game loop** (editor remains responsive), and shows a blocking toast with “Resume/Reload”.
- `fatal` → host triggers `on_unload` and unloads the module; the editor switches to a clean state.

### 0.8 Vulkan on macOS (MoltenVK) — required flags

Amend the Renderer Day‑1 checklist:

- Create the instance with `VK_INSTANCE_CREATE_ENUMERATE_PORTABILITY_BIT_KHR` and enable `VK_KHR_portability_enumeration`.
- When selecting the device on macOS, enable `VK_KHR_portability_subset`.
- Prefer Vulkan **1.3** features where available (or enable `VK_KHR_dynamic_rendering` + pipeline rendering info).

### 0.9 `build.zig` portability & install

- Add per‑platform linking (Windows: copy `game.dll` next to the editor EXE; Linux: `.so` to `zig-out/bin`; macOS: `.dylib` and ensure frameworks on `@rpath`).
- Add an **install step** that places the freshly built game library beside the editor so hot‑reload picks it up immediately.
- On Windows, load a **shadow copy** (timestamped filename) to avoid file locks while rebuilding.

### 0.10 Ship mode build shape

Introduce `-Dship=true`:

- Editor features and hot‑reload **disabled**.
- Either **statically link** the game into a single EXE, or ship a single `game` DLL with reload paths compiled out.
- Enable LTO/strip and deterministic fixed‑step option; bake pipeline caches.

### 0.11 Minor ABI guidance

- Prefer **integer handles** at the ABI boundary for long‑lived objects; keep raw pointers only for **transient, frame‑local** data.
- Keep **counts/lengths/offsets as `u64`** in ABI types; keep **size/version fields as `u32`**; maintain 8-byte struct alignment at extern boundaries.

---

> Implementation note: update the existing `api.zig` to reflect `JobDesc.owner_tag`, add `asset_acquire/asset_release`, and wire quiesce to filter by `JOB_OWNER_GAME`. Update the renderer init path for macOS portability flags and extend `build.zig` with per‑OS copy/rpath steps.

---

## 1) Vision & Scope

- **Goal:** A lean, extensible engine and editor suitable for indie‑scale games and technical demos, emphasizing hot reload, fast iteration, cross‑platform reach (Win/Linux/macOS), and a modern renderer (Vulkan).
- **Non‑Goals (initially):** Full console support, networked multiplayer, AAA content tooling. These can be added later behind stable interfaces.

---

## 2) High‑Level Architecture

### 2.1 Layers

1. **Editor/Launcher (host EXE):** Owns process, lifetime, allocators, and all systems. Runs the main loop, UI, asset browser, etc.
2. **Engine Core & Plugins (static or dynamic libs):** Window/input, renderer, physics, audio. Exposed to game via opaque pointers and function tables.
3. **Game Module (shared library):** Hot‑reloadable `game` `.dll/.so/.dylib` exposing a `GameAPI` function table. Contains game state & logic only.

### 2.2 Ownership & Memory

- **Editor/Launcher owns everything.** Game code receives non‑owning pointers via `EngineContext` and service function pointers.
- Use **arena + frame allocators** in the Editor; subsystems get scoped allocators; avoid hidden ownership.

### 2.3 Threading Model

- **Main thread:** window messages, input collection, ImGui, high‑level orchestration, kicking render frames.
- **Workers (N = cores − 1):** job system for asset IO, decompression, culling, animation, etc.
- **Renderer thread (optional):** submit work & manage GPU timelines.

---

## 3) Project Structure

```
ZigEngine/
├─ build.zig
├─ build.zig.zon
├─ assets/                      # raw assets (source)
├─ cooked/                      # processed/bundled assets
├─ src/
│  ├─ editor/                   # host EXE (owns systems)
│  │  └─ main.zig
│  ├─ core/                     # logging, allocs, time, filewatch, hot-reload
│  ├─ engine/                   # public engine interfaces
│  ├─ runtime/                  # ECS, scene, serialization
│  ├─ plugins/
│  │  ├─ window_sdl3/
│  │  ├─ renderer_vulkan/
│  │  ├─ audio_miniaudio/
│  │  ├─ physics_backend/       # Jolt (custom C shim) + Box2D (C API)
│  │  └─ ui_imgui/
│  └─ game/                     # hot‑reloadable shared lib
│     └─ lib.zig
├─ tools/
```

- Platform names for the game module: `game.dll` (Windows), `libgame.so` (Linux), `libgame.dylib` (macOS).
- Add an install step that copies the freshly built game module next to the editor binary.
- Windows: implement a **shadow‑copy** helper that copies `game.dll` to a timestamped filename before load to avoid file locks.
- macOS: when using MoltenVK via frameworks/bundles, ensure the framework is on `@rpath` so the loader can find it at runtime.

---

## 4) Build & Tooling

- Build using Zig; dependencies are vendored. Use `volk` for Vulkan loader.
- **Targets:** Win64, Linux x86\_64, macOS arm64/x86\_64 (MoltenVK for macOS).
- **Artifacts:**
  - `editor` (EXE)
  - `game` (shared lib)
  - `cooker` (CLI)

### 4.1 Build Modes

- Debug, ReleaseSafe, ReleaseFast, Ship (`-Dship=true`).
- Pin the Zig toolchain version in the repo (README/CI) and state the tested version to avoid drift.
- CI matrices across platforms; linting with `zig fmt --check`.

### 4.2 build.zig install (per-OS snippet)

> Note: This snippet targets the current Zig Build API; if your Zig version differs, adjust `install`/`lookup` calls accordingly.

```zig
// build.zig (snippet) — installs the game library next to the editor binary
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const editor = b.addExecutable(.{
        .name = "editor",
        .root_source_file = .{ .path = "src/editor/main.zig" },
        .target = target,
        .optimize = optimize,
    });

    const game = b.addSharedLibrary(.{
        .name = "game",
        .root_source_file = .{ .path = "src/game/lib.zig" },
        .target = target,
        .optimize = optimize,
    });

    // Install both artifacts to the prefix
    b.installArtifact(editor);
    b.installArtifact(game);

    // Copy the game library beside the editor binary so hot-reload picks it up.
    const lib_name = if (target.result.os.tag == .windows) "game.dll"
        else if (target.result.os.tag == .macos) "libgame.dylib"
        else "libgame.so";

    const copy_game_next_to_editor = b.addInstallFileWithDir(
        game.getOutputSource(), .bin, lib_name,
    );
    copy_game_next_to_editor.step.dependOn(&editor.step);

    // RPATH so the editor can find adjacent libs at runtime
    if (target.result.os.tag == .linux) {
        editor.addRPath("$ORIGIN");
    } else if (target.result.os.tag == .macos) {
        editor.addRPath("@loader_path");
        // If using MoltenVK as a framework, ensure it's on @rpath too:
        // editor.addRPath("@executable_path/../Frameworks");
        // editor.linkFramework("MoltenVK");
    }
}
```

---

## 5) Stable C ABI & Hot Reload

### 5.1 ABI philosophy

- Freeze a small C ABI surface; evolve via **version + size + caps bitset**.
- Only pass POD, opaque pointers, and function pointers. No Zig slices/allocators across the boundary.
- All memory is owned by the host; the game never frees host memory.
- **Targets are 64‑bit only**; **counts/lengths/offsets** are **u64**; **struct size/version fields** (e.g., `EngineContext.size`, `GameAPI.size`) are **u32**.
- **Alignment:** all extern structs are padded to an 8-byte multiple; the host **zero-fills `_reserved`** fields.
- **Endianness:** ABI uses platform‑native endianness (little‑endian on supported targets). Big‑endian is **unsupported**.

#### 5.1.1 Memory ownership & allocation (Normative)

- **Global rule:** The **host/editor owns memory** for engine subsystems; the game never frees host memory.
- **Across ABI:** No allocators cross the boundary; only POD, integer handles, and function pointers.
- **Temp/scratch:** `temp_alloc/temp_free` are **frame‑local**; pair within the same frame. Suggested per‑allocation cap: **256 KiB**.
- **Assets:** Host owns asset memory; game receives `AssetHandle` IDs and must `asset_acquire/asset_release`.
- **ECS:** Host owns the Flecs world and component storage. The game sees an opaque `*Scene` and iterates via handle‑based queries.
- **SoA bridge:** The **host allocates and owns** `Transforms` SoA buffers (aligned, persistent); the physics plugin writes into them **in place**.
- **Physics plugin:** Owns the Jolt/Box2D worlds and their allocations (plugin‑local arena or supplied `JInit.sys_alloc`). Game code **never frees** physics memory.
- **Query chunks:** `QueryChunkOut.base` is valid **until the end of the current dispatch/frame**; do **not** retain beyond the frame.

### 5.2 Cross‑ABI types

```zig
pub const GameResult = enum(c_int){ ok=0, recoverable=1, fatal=-1 };
pub const SpanU8 = extern struct { ptr: [*]u8, len: u64 }; // lifetime: valid until end of current frame unless noted
pub const ReadOnlySpanU8 = extern struct { ptr: [*]const u8, len: u64 }; // convenience alias for const data
pub const SpanU8Cap = extern struct { ptr: [*]u8, len: u64, cap: u64 }; // input: cap=capacity, len ignored; output: len=bytes written
pub const Guid = [16]u8; // 16-byte GUID; textual form 8-4-4-4-12 lowercase hex
pub const QueryHandle = u64;
pub const QueryChunkOut = extern struct { base: ?*anyopaque, count: u64, stride: u64 }; // engine-defined packed chunk; `stride` is bytes; `base` + `stride` let you walk items
```

**Text across ABI:** never use `SpanU8` for strings; always use `ReadOnlySpanU8` (immutable) for text.

### 5.3 Structs & function table

```zig
// src/engine/api.zig

pub const JobFn = *const fn(*anyopaque) callconv(.C) void;
pub const JobDesc = extern struct { fn_ptr: JobFn, user: *anyopaque, owner_tag: u32, _pad: u32 };

pub const EngineServices = extern struct {
    log: ?*const fn(ReadOnlySpanU8) callconv(.C) void,

    // Job system
    schedule_job: ?*const fn(JobDesc) callconv(.C) void,

    // Assets — returns a handle; caller must release.
    asset_get_by_guid: ?*const fn(*const Guid) callconv(.C) ?AssetHandle,
    asset_acquire:     ?*const fn(AssetHandle) callconv(.C) void,
    asset_release:     ?*const fn(AssetHandle) callconv(.C) void,

    // ECS/queries (opaque)
    // ECS query iterators (handle-based)
    ecs_query_open:  ?*const fn(ReadOnlySpanU8) callconv(.C) ?QueryHandle,
    ecs_query_next:  ?*const fn(QueryHandle, *QueryChunkOut) callconv(.C) bool,
    ecs_query_close: ?*const fn(QueryHandle) callconv(.C) void,
};
pub const EngineContext = extern struct {
    ctx_magic: u32,         // CTX\0 = 0x43545800
    ctx_revision: u16,      // bump on wire/layout changes for EngineContext
    _ctx_pad: u16,          // keep 8-byte alignment

    size: u32,              // @sizeOf(EngineContext) in this build
    caps: u64,              // capability bits (see 5.5)
    build_id: u64,          // engine build hash/ID for logging/compat checks
    _reserved: [32]u8,      // padding for future fields (host zero-fills)

    services: ?*EngineServices,

    window:   ?*anyopaque,
    renderer: ?*anyopaque,
    physics:  ?*anyopaque,
    audio:    ?*anyopaque,
    event_bus:?*anyopaque,
    asset_mgr:?*anyopaque,
    job_sys:  ?*anyopaque,

    // Optional temp allocators for short‑lived cross‑ABI buffers
    temp_alloc: ?*const fn(u64) callconv(.C) ?*anyopaque,
    temp_free:  ?*const fn(*anyopaque) callconv(.C) void,
};

pub const Scene = opaque {};
pub const AssetHandle = u64;

pub const GameAPI = extern struct {
    // hard fail fast on mismatched binaries
    abi_magic: u32,        // 'GAME' = 0x47414D45
    abi_revision: u16,     // bump on wire layout tweaks even if size is same
    _pad: u16,             // keep subsequent fields 32-bit aligned

    version: u32,          // semantic break counter for this API surface
    size: u32,             // @sizeOf(GameAPI)
    caps: u64,             // optional features supported by this game module

    on_load:          ?*const fn(*EngineContext, bool)        callconv(.C) GameResult,
    on_update:        ?*const fn(*Scene, f32)                 callconv(.C) GameResult,
    on_unload:        ?*const fn()                            callconv(.C) void,

    // Optional hot‑reload state persistence (behind a cap bit)
    on_reload_save:    ?*const fn(*SpanU8Cap)                    callconv(.C) void,  // writes a small state blob into a host buffer
    on_reload_restore: ?*const fn(ReadOnlySpanU8)                callconv(.C) void,  // rehydrates after load
};
```

**Timestep:** `on_update(*Scene, dt)` uses seconds (f32). Physics runs on a fixed step; game logic may be fixed or variable.

### 5.4 Hot‑reloader (Editor)

```zig
// src/core/hot_reload.zig (excerpt)
pub fn quiesce() void {
    // stop scheduling new jobs from game; wait for game‑tagged jobs to finish; drain event queues
}
```

Reload flow: detect file change → **quiesce** → `on_reload_save` → unload old → load new → `on_load(ctx, true)` → `on_reload_restore` (if supported).

**Thread quiesce semantics (normative)**

- Stop accepting new jobs tagged `game`.
- Wait for all `game` jobs to finish (configurable watchdog, e.g., 2–5s) else keep the **old module active**, and surface a **blocking error** in the editor.
- Drain event queues targeting game callbacks.
- Block input dispatch to the game until reload completes.
- Game must unsubscribe all EventBus callbacks in `on_unload`.

**Reload survival budget**

- Default state blob budget: **128 KB**. The game may request a bigger buffer in `on_load`; the host may grant up to a platform cap.
- Host passes a `SpanU8Cap` to `on_reload_save` where `cap` is the maximum bytes. The game must write ≤ `cap` and set `len` to bytes written.
- Host zero-initializes `len`; the game sets it to the number of bytes written; the host reads exactly `len` bytes on restore.
- If the granted size is insufficient or versions mismatch, **fall back to a clean load** and log a telemetry counter.
- Host verifies blob size/hash in Debug and logs warnings in ReleaseSafe; never crash on restore failure.

**Resource fences**

- Insert a **GPU timeline fence** and wait for in‑flight frames that reference game‑owned resources before unloading the DLL.
- Resources created by the game (GPU/CPU) must be destroyed or **handed back** to the host with clear ownership before unload; the host asserts no leaked resources during reload and logs offenders.

### 5.5 Capability bits (examples)

- bit 0: supports `on_reload_save/restore`
- bit 1: uses engine job system
- bit 2: uses event bus
- bit 3: requires fixed‑timestep updates

**Caps ownership**

- `EngineContext.caps` = **host-provided** capabilities.
- `GameAPI.caps` = **game-reported** features/requirements.
- On load, the host validates compatibility (e.g., if the game requires cap that the host lacks it → **MUST hard-fail `on_load` with a clear error**).

```zig
// src/engine/api.zig — shared capability bit constants
pub const CAP_GAME_SUPPORTS_RELOAD_STATE: u64 = 1 << 0; // uses on_reload_save/restore
pub const CAP_GAME_USES_JOBS:              u64 = 1 << 1; // schedules engine jobs
pub const CAP_GAME_USES_EVENTBUS:          u64 = 1 << 2; // subscribes/publishes events
pub const CAP_GAME_FIXED_TIMESTEP:         u64 = 1 << 3; // requires fixed dt
```

---

## 6) Input, Windowing, & Time

- SDL3 or native backends; raw‑input path on Windows; hi‑DPI scaling; relative mouse controls.
- Frame timing from high‑res timers; fixed/variable timestep support.

---

## 7) Event Bus (lock‑light)

- **Goal:** decoupled messaging with predictable lifetimes.
- **Design:** per‑kind double buffer; thread‑safe publish (MPSC); main‑thread dispatch.
- **Handles:** `{ index, generation }` to prevent use‑after‑free.

```zig
// src/core/event_bus.zig
const std = @import("std");
```

---

## 8) Job System

- **Phase 1:** thread pool + MPSC; `schedule`, `parallelFor`, `wait`.
- **Phase 2:** work‑stealing deques; fiber/wait‑groups.
- **Frame fences:** APIs that touch frame‑lifetime data require a fence and are signaled on completion; editor shows a red channel if a fence isn’t signaled by frame end.
- **Long‑running jobs:** jobs can mark themselves as long‑running to move to a separate queue and avoid starving short tasks.
- **Worker‑safe APIs:** document which engine APIs are callable from workers; others assert on background usage.

```zig
// src/core/jobs.zig (sketch)
pub const JobHandle = struct { id: u32, gen: u32 };
pub const JobFn = *const fn(*anyopaque) callconv(.C) void;
pub const ForFn = *const fn(u32, *anyopaque) callconv(.C) void;
pub const JobDesc = api.JobDesc; // alias to avoid drift
```

---

## 9) Renderer (Vulkan first)

- Primary backend Vulkan; RenderDoc/Tracy integration.
- **Phase 1:** Immediate mode using Dynamic Rendering; per‑frame transient resources; basic forward pass.
- **Phase 2:** Render Graph (passes/resources, automatic barriers), timestamps, descriptor management, staging uploads, transient pools.
- **Shading:** GLSL or HLSL → SPIR‑V (toolchain integrated in cooker).
- **macOS:** via MoltenVK.

#### 9.2.1 Day‑1 checklist

- Dynamic Rendering, one graphics queue, triple buffering.
- Robust swapchain recreation (resize, minimize, format change).
- Per‑frame transient arenas for command pools, descriptor sets, and staging buffers; reset every frame.
- Frame pacing with CPU/GPU timestamps and Tracy markers.
- Headless/offscreen path for CI (no window).
- Shader hot‑reload with pipeline cache warming and graceful fallback on compile failure.
- **Shader reflection metadata** baked into cooked assets (bindings, push constants, specialization) so hot‑reload doesn’t require code changes.
- **Shader compilers:** support **DXC** (HLSL) and **glslang** (GLSL). Always run **SPIR‑V validation** (SPIR‑V Tools) before pipeline rebuild.
- **Device loss playbook:** detect VK\_ERROR\_DEVICE\_LOST → sleep/backoff → full device + swapchain recreate → telemetry increment.

#### 9.2.2 Device abstraction & WebGPU fallback

- Keep a minimal `IDevice` interface (queues, swapchain, buffer/tex creation, shader/pipeline creation).
- Primary path: Vulkan implementation.
- Optional fallback: `wgpu-native`/WebGPU implementation behind the same interface for platforms where Vulkan support is fragile.
- Decision checklist: if you need web builds, simpler portability, or tooling limits block you, consider enabling the WebGPU backend.

### 9.3 UI/Debug (Dear ImGui)

- Docking, viewport window, log console, metrics/profiler panels, gizmos (ImGuizmo via C shim if needed).

### 9.4 Audio (miniaudio)

- Init device, play/stop sounds, streaming music, basic mixer/bus with volume.

## 9.5 Physics Backend (Jolt) — Integration Plan

**Choice:** Custom, tiny C shim over Jolt’s C++ API (static lib). Minimal surface, batching-friendly, allocator-aware, and future-proof.

### 9.5.1 C shim (header sketch)

```c
// jolt_shim.h
#ifdef __cplusplus
extern "C" {
#endif
#include <stdint.h>
typedef uint32_t JHandle; // index|generation

typedef struct { float x,y,z; } JVec3;
typedef struct { float x,y,z,w; } JQuat;

typedef struct {
  int   type;        // 0=BOX,1=SPHERE,2=CAPSULE,... (engine enum mirrored)
  JVec3 half_extents;
  float radius;
  float height;
  float density;
  float friction;
  float restitution;
} JShapeDesc;

typedef struct {
  JVec3 gravity;
  uint32_t max_bodies, max_shapes, max_joints;
  uint32_t worker_threads;
  void* (*sys_alloc)(size_t); // optional custom allocators (nullable)
  void  (*sys_free)(void*);
} JInit;

typedef struct {
  float* pos_xyz;   // len = 3*count
  float* rot_xyzw;  // len = 4*count
  float* lin_v;     // len = 3*count
  float* ang_v;     // len = 3*count
  uint64_t count;
} JTransforms;

typedef enum {
  J_STATUS_OK = 0,
  J_STATUS_ERR_CAPACITY = -1,
  J_STATUS_ERR_INVALID  = -2,
  J_STATUS_ERR_UNKNOWN  = -3,
} JStatus;

JStatus j_init(const JInit*);
void j_shutdown(void);

JStatus j_bodies_create_batch(const JShapeDesc* shapes, uint64_t n,
                               const JTransforms* x0,
                               JHandle* out_handles, uint64_t* out_created);
JStatus j_bodies_destroy_batch(const JHandle* handles, uint64_t n,
                               uint64_t* out_destroyed);

JStatus j_step(float dt_sec, JTransforms* io_bodies);
// Optional: batched ray/overlap
#ifdef __cplusplus
}
#endif
```

### 9.5.2 Zig physics plugin API (host side)

Expose only **coarse-grained** C ABI:

```zig
pub const PhysicsInitDesc = extern struct {
    gravity: [3]f32,
    max_bodies: u32,
    max_shapes: u32,
    max_joints: u32,
    worker_threads: u32,
};

pub const PhysStatus = enum(c_int){ ok=0, capacity=-1, invalid=-2, unknown=-3 };

pub extern fn physics_init(desc: *const PhysicsInitDesc) PhysStatus;
pub extern fn physics_shutdown() void;

pub const BodyHandle = u32;
pub const GEN_MASK: u32 = 0xFF00_0000; // 8-bit generation
pub const IDX_MASK: u32 = 0x00FF_FFFF; // 24-bit index (~16.7M bodies)
pub inline fn bodyIdx(h: BodyHandle) u32 { return h & IDX_MASK; }
pub inline fn bodyGen(h: BodyHandle) u32 { return (h & GEN_MASK) >> 24; }
// Note: If you need >16.7M bodies or >255 generations, adjust the bit layout accordingly.

pub extern fn physics_create_bodies_batch(/* ShapeDesc[] */, *const Transforms, out_handles: [*]BodyHandle, capacity: u64, out_created: *u64) PhysStatus;
pub extern fn physics_destroy_bodies_batch(handles: [*]const BodyHandle, count: u64, out_destroyed: *u64) PhysStatus;
pub extern fn physics_step(dt_sec: f32, bodies: *Transforms) PhysStatus; // in/out SoA
```

### 9.5.3 Error handling & semantics
- All functions return **`PhysStatus/JStatus`**: `ok=0`, `capacity=-1` (not enough room), `invalid=-2` (bad handle/args), `unknown=-3`.
- **Create/destroy (batch):** on capacity, the call may **partially succeed**. The number of created/destroyed items is written to the corresponding `out_*` pointer; the status is `capacity`.
- **Step:** returns `ok` on success; `invalid` for bad buffers; `unknown` for internal errors.
- **Policy:** Never crash from the plugin; report via status. The editor translates non‑`ok` statuses into UI toasts and logs details.

### 9.5.4 Jolt build checklist (first‑build hints)
- Build **static** with `-O3 -flto -fno-exceptions -fno-rtti` (PIC on Linux/macOS). If using CMake, set `CMAKE_POSITION_INDEPENDENT_CODE=ON`.
- Use the platform C++ toolchain (MSVC on Windows; clang++ on Linux/macOS). Zig links it via `editor.linkLibCpp()`.
- Ensure consistent runtime on Windows (`/MD` or `/MT`—pick one across all static libs). Prefer `/MD` for editor builds.
- Define any required platform macros (e.g., `NOMINMAX` on Windows) in the shim build if needed.
- Vendor a known‑good Jolt commit; add a simple unit test that verifies `j_init()` and a no‑op `j_step()` on all platforms in CI.
```zig
pub const Transforms = extern struct {
    pos:   [*]f32, // x,y,z (len = 3*count)
    rot:   [*]f32, // x,y,z,w (len = 4*count)
    vel:   [*]f32, // vx,vy,vz
    ang:   [*]f32, // wx,wy,wz
    count: u64,
};
```

- **Alignment:** buffers should be at least 16-byte aligned (32 preferred) for SIMD on x86\_64/arm64.
- These buffers are **owned by the host**, preallocated, and reused each frame. No per-frame heap churn.
- The physics plugin updates them **in place**.


## 10) Assets & Cooker

- **Assets:** source in `assets/`, processed to `cooked/` with GUIDs and stable type IDs.
- **Cooker:** CLI that imports, validates, and converts to engine-ready formats (textures, meshes, shaders). Emits reflection metadata where needed (e.g., shaders).
- **Hot‑reload:** file watch → cook changed assets → notify engine; engine swaps in new versions safely between frames.
- **Versioning:** assets carry format/version; loader upgrades or rejects with a clear error.

## 11) ECS & Scene

- **World ownership:** the **host/editor owns the Flecs world** and all component storage. The game sees an opaque `*Scene`.
- **Query API (handle‑based):** `ecs_query_open/next/close` return engine‑defined packed chunks (`QueryChunkOut`) whose `base` is valid **for this frame/dispatch only**.
- **SoA bridge:** gather `[Transform, RigidBody]` into contiguous, aligned SoA buffers; call `physics_step`; scatter results back in one tight loop.
- **Threading rules:** add/remove on the main thread; read‑only queries may run on workers if documented. Never cache raw component pointers across frames or reloads.
- **Entity ↔ Body mapping:** maintain `Entity -> BodyHandle` (and optional `BodyHandle -> SoA index`) maps; swap‑erase on removal to keep SoA dense.

---

## 12) Serialization

- Binary scene format with versioning; JSON debug exporter.
- Save/load pipelines; diffs for hot‑reload.

---

## 13) Telemetry & Metrics

- Frame time, draw calls, VRAM usage, asset load counts; hot‑reload counts (soft/hard), reload fallback ratio.
- Editor panels showing timings, job queues, frame memory.

### 13.1 Job System Observability

- Live per‑worker queue depth, task latency histogram, and a "quiesce" indicator.
- Editor panel to pause/resume workers and change worker count in Debug.

### 13.2 Crash Reporting

- Minidumps on crash; symbol upload step in CI; "Open last crash" button in editor.
- Include build ID from `EngineContext` in reports.

### 13.3 Record & Replay

- Capture input/events for 10–30s and replay deterministically.
- Seed a fixed RNG for game code during replay to keep behavior deterministic.
- Used in perf triage and to repro reload issues.

---

## 14) Platform Notes

- **Windows:** load a uniquely named **shadow copy** of the DLL to avoid file locks; prefer `SetDefaultDllDirectories(LOAD_LIBRARY_SEARCH_DEFAULT_DIRS)` and use `LoadLibraryEx` with safe flags.
- **Linux:** prefer `.so` version suffix or copy to a temp filename and atomically swap; keep the old handle open until the new one is live; inotify for file watch.
- **macOS:** `.dylib`; code signing for distribution; kqueue for file watch; MoltenVK; note sandboxing requirements for file access.
- **General:** avoid in‑place overwrites; always write to a new filename and swap; resolve `game` from the project's build output directory; no PATH search.

---

## 15) Conventions (Zig)

- Modules/namespaces `snake_case`, types `PascalCase`, functions `camelCase`, files `snake_case.zig`.
- Public symbols prefixed with module names; no `pub` leakage across internal modules.
- Error handling via `!Error` where appropriate; use `defer`/`errdefer` for cleanup.
- Logging via central logger; severity levels; convert to `LogError` events for UI.
- **Allocators:** global general‑purpose; per‑frame arena; long‑lived arenas for subsystems; **no allocator across ABI**.
- **File Watch:** polling fallback (mtime) with platform fast paths (inotify/ReadDirectoryChangesW/kqueue).

---

## 16) Roadmap (Example Milestones)

- W1—Bring up window/input, basic swapchain, hot‑reload scaffold.
- W2—Jobs v1, EventBus v1, asset pipeline skeleton, shader compilation.
- W3—Forward renderer pass, texture/mesh loading, ImGui overlay.
- W4—ECS v1, simple scene, editor dock, play/stop.

---

## 17) Quality Gates (CI)

- **Build matrix:** Debug, ReleaseSafe, ReleaseFast across Win/Linux/macOS.
- **Static analysis:** `zig fmt --check`, `zig build test`, optional clang‑tidy for C glue.
- **Runtime tests:** headless render, asset import smoke tests.
- **ABI stress:** open/close `game` 1000×; ensure callbacks fully detach.
- **ABI Guards:** attempt to load deliberately mismatched modules and assert clean failure before any calls.

---

## 18) Security, Robustness, Risks

- Validate all external inputs (assets, shaders).
- Sandboxed cooker for untrusted content.
- Watch for allocator misuse; guard pages in Debug.

---

## 19) Editor Skeleton (example)

### 19.1 Editor `main.zig` (skeleton)

```zig
const std = @import("std");
const api = @import("../engine/api.zig");
const HR = @import("../core/hot_reload.zig");

fn hostLog(msg: api.ReadOnlySpanU8) callconv(.C) void {
    if (msg.ptr != null and msg.len > 0) {
        const slice = msg.ptr[0..msg.len];
        std.debug.print("{s}
", .{slice}); // fixed newline
    }
}

fn resolveGameModulePath() []const u8 {
    // TODO: return OS-specific path, e.g., zig-out/bin/game.dll on Windows,
    // zig-out/lib/libgame.so on Linux, zig-out/lib/libgame.dylib on macOS
    return "./zig-out/lib/libgame.so";
}

pub fn main() !void {
    // init window, renderer, subsystems...
    var ctx: api.EngineContext = .{
        .ctx_magic = 0x43545800, .ctx_revision = 1, ._ctx_pad = 0,
        .size = @sizeOf(api.EngineContext), .caps = 0, .build_id = 0,
        ._reserved = .{0} ** 32,
        .services = null,
        .window = null, .renderer = null, .physics = null, .audio = null,
        .event_bus = null, .asset_mgr = null, .job_sys = null,
        .temp_alloc = null, .temp_free = null,
    };

    // minimal EngineServices so game can log during on_load
    var services = api.EngineServices{
        .log = hostLog,
        .schedule_job = null,
        .asset_get_by_guid = null,
        .asset_acquire = null,
        .asset_release = null,
        .ecs_query_open = null,
        .ecs_query_next = null,
        .ecs_query_close = null,
    };
    ctx.services = &services;

    var hr = HR.HotReloader{};
    try hr.load(resolveGameModulePath());
    if (hr.game) |g| if (g.on_load) |f| _ = f(&ctx, false);

    var last = std.time.milliTimestamp();
}
```

---

## 20) Shipping Checklist

- Rebuild in **Ship mode** (LTO/strip), bake pipeline caches, disable validation layers.
- Asset manifest freeze + hashes; crash reporter configured.

---

## 21) What’s Next

- Generate starter repo with this layout and stubs so you can `zig build` immediately.
- Add `plugins/physics_backend` with `jolt_shim.h/.cpp` and a Zig wrapper exposing the coarse C ABI (`physics_init/step/create/destroy`).
- Bring up Flecs C import and ECS table iteration; wire the SoA bridge (`Transforms`) to the physics step.
- Add CI: build on Win/Linux/macOS + headless smoke tests (load game, tick, hot-reload).
- Implement the Windows shadow-copy loader helper for the game DLL.

