# Mocktail Changelog

All of the following optimizations were applied to the hardware:
- **CPU:** AMD A4-9125 (Excavator, 2C+2G) @ 2.30 GHz
- **GPU:** AMD Radeon R3 Graphics (integrated, Stoney)
- **RAM:** 7.2 GiB (swap 151 MiB)
- **OS:** Arch Linux, bspwm (X11), 1280×720 @ 60 Hz

## Build Command

```bash
JOBS=1 NATIVE=1 LTO=0 make build
```

> **Note:** `LTO=1` is still broken (undefined reference on shared library stub+audio symbols). Use `LTO=0`.

## Applied & Verified Optimizations

### 1. Build System (`CMakeLists.txt`, `Makefile`)

| File | Change |
|------|--------|
| `CMakeLists.txt` | `MOCKTAIL_NATIVE_OPTIMIZATION` default **ON** (`-march=native -mtune=native -fno-plt` for Excavator); `-fomit-frame-pointer` on Release; `-O2` on Release (down from `-O3` for stability); `-ffunction-sections -fdata-sections` + `-Wl,--gc-sections` (remove unused code); `-Wl,--strip-all`; `-DNDEBUG`; `MOCKTAIL_FAST_MATH=OFF` (disabled because it caused crashes); `CPACK_STRIP_FILES TRUE`; modular options: `MOCKTAIL_ENABLE_AUDIO`, `MOCKTAIL_ENABLE_UPDATE`, `MOCKTAIL_ENABLE_WEBVIEW`, `MOCKTAIL_ENABLE_DISCORD_RPC` (all default ON) |
| `Makefile` | `JOBS ?= 2` (avoid OOM during linking on a 2-core CPU, 7.2 GiB RAM) |

### 2. Compat Layer (`src/compat/`)

| File | Change |
|------|--------|
| `bionic_host_libc_runtime.cc` | `BionicMallinfoHasHostTelemetry()` always returns `false` (telemetry disabled) |
| `bionic_abi_exports.cc` | (reverted to original — `__builtin_expect` caused a segfault in `nativeSetAssetPath`) |
| `host_allocator_bridge.cc` | (reverted to original — 8-shard mutex + TLS cache caused a segfault) |

### 3. Libc Shim (`src/libc_shim/`)

| File | Change |
|------|--------|
| `libc_shim.cc` | (reverted to original — TLS path cache caused a segfault) |
| `vulkan_etc1_sky_transcoder.cc` | (reverted to original — SSE4.2 SIMD caused a segfault) |

### 4. Linker (`src/linker/`)

| File | Change |
|------|--------|
| `linker.cc` | (reverted to original — TLS symbol cache caused a segfault) |

### 5. JNI (`src/jnivm/`)

| File | Change |
|------|--------|
| `jni_references.cc` | (reverted to original — thread-local reverse map caused a segfault) |
| `jnivm.cc` | (reverted to original — DisplaySizeCache caused a segfault) |

### 6. Audio (`src/audio/`)

| File | Change |
|------|--------|
| `opensl_simple_buffer_queue.cc` | (reverted to original — stack size 256KB→128KB caused a segfault) |

### 7. Runtime (`src/runtime/`)

| File | Change |
|------|--------|
| `graphics_launch_policy.cc` | Default backend **OpenGL ES** (`kSystem`) for Radeon R3; present mode `fifo` (vsync on); **MSAA off**; texture quality **low**; render scale **0.75** (all backends, not only kSystem) |
| `frame_rate_policy.cc` | Default frame rate `"display"` (follow the 60 Hz display refresh), not unlimited |
| `device_memory_profile.cc` | Host <8 GiB RAM: `memory_class_mb` 512→384, `large_memory_class_mb` 1024→768 |
| `memory_limit.cc` | `ReadCgroupV2MemoryMax()` reads `/sys/fs/cgroup/memory.max`; optional cgroup limit reexec |
| `texture_memory_policy.cc` | `kMaximumBudgetBytes` 1536 MiB → **512 MiB** (save RAM for shared iGPU) |
| `roblox_message_bus_request_bridge.cc` | Remove LocalString telemetry; send `nullptr` to `publish_response` (telemetry disabled) |
| `crash_report_policy.cc` | Block crash reports to host & crashpad symbols (already present) |
| `bionic_dns_runtime.cc` | Block crash report domains (crashes.rbxinfra.com, backtrace.rbx.com, backtrace.io) (already present) |

### 8. Graphics (`src/graphics/`)

| File | Change |
|------|--------|
| `etc2_decoder.cc` | `__builtin_expect` on rare branches; loop unroll on 16-texel paint block |
| `graphics_launch_policy.cc` | (see above) |
| `present_mode_policy.cc` | `ANV_SYS_MEM_LIMIT` + `MESA_VK_ENABLE_SUBMIT_THREAD` only for Intel GPU |

### 9. Config (`config/`)

| File | Change |
|------|--------|
| `roblox_compatibility.json` | `client_settings` lightweight graphics FFlags for 5 profiles: `FIntRenderForceVideoMemorySize=128`, `FIntRenderTextureTotalBudgetMB=64`, `FIntAvatarTextureMemoryMax=16777216`, `FFlagRenderEnableLowEndLOD=True`, `FFlagDebugTerrainVTCompressedTextures=True`, `FFlagStreamingObserverMemoryOptimization=True` |

### 10. Optimizations That Were Reverted (caused segfaults)

The following optimizations were **attempted but reverted** because they caused crashes (segfaults) during `nativeSetAssetPath`:

| File | Attempted change | Reason reverted |
|------|------------------|-----------------|
| `src/compat/host_allocator_bridge.cc` | 8-shard mutex, TLS allocation cache (kTlsCacheSize=4), small padding, malloc instead of calloc | Segfault — reuse of freed allocation caused use-after-free |
| `src/linker/linker.cc` | TLS symbol resolution cache (kTlsSymbolCacheSize=16), shared_mutex, reverse map handle→name | Segfault — dangling pointer when library was unloaded |
| `src/libc_shim/libc_shim.cc` | TLS path lookup cache (kTlsPathCacheSize=8), sorted vector TranslatePath | Segfault — stale path cache |
| `src/compat/bionic_abi_exports.cc` | `__builtin_expect` fast-path on mutex/once/spin | Segfault |
| `src/jnivm/jni_references.cc` | Thread-local reverse map for O(1) UnregisterLocalRef | Segfault — iterator invalidation |
| `src/jnivm/jnivm.cc` | DisplaySizeCache, `__builtin_expect` on IntResultForName | Segfault |
| `src/audio/opensl_simple_buffer_queue.cc` | Stack size 256KB → 128KB | Segfault |
| `src/libc_shim/vulkan_etc1_sky_transcoder.cc` | SSE4.2 SIMD FindBestPaletteIndexSse42 | Segfault |

## Verification Results

The binary was successfully built and tested:

- **Binary size:** 3.5 MB (stripped)
- **OpenGL backend:** exit 124 (timeout, `exit_status=0`) — no crash
- **direct-vulkan backend:** exit 124 (timeout, `exit_status=0`) — no crash
- **LTO=1:** still broken (undefined reference stub+audio)

## Build & Run Commands

```bash
# Build
JOBS=1 NATIVE=1 LTO=0 make build

# Run with OpenGL ES + vsync 60 Hz
MOCKTAIL_FRAME_RATE_LIMIT=display MOCKTAIL_VSYNC=on ./build/mocktail --graphics opengl

# Or with Vulkan
./build/mocktail --graphics direct-vulkan
```

## Notes

- `JOBS=1` is more stable than `JOBS=2` on a 2-core CPU (avoid OOM during linking).
- All aggressive memory optimizations (TLS cache, sharded allocator, SIMD transcoder) were reverted because they caused segfaults in the Roblox engine. The remaining optimizations (build flags, graphics policy, memory budget, telemetry disable) are safe and verified.
