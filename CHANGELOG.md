# Mocktail Changelog

Semua optimasi berikut diterapkan untuk hardware:
- **CPU:** AMD A4-9125 (Excavator, 2C+2G) @ 2.30 GHz
- **GPU:** AMD Radeon R3 Graphics (integrated, Stoney)
- **RAM:** 7.2 GiB (swap 151 MiB)
- **OS:** Arch Linux, bspwm (X11), 2× 1280×720 @ 60 Hz

## Build Command

```bash
JOBS=1 NATIVE=1 LTO=0 make build
```

> **Catatan:** `LTO=1` masih rusak (undefined reference pada simbol shared library stub+audio). Gunakan `LTO=0`.

## Optimasi yang Diterapkan & Terverifikasi

### 1. Build System (`CMakeLists.txt`, `Makefile`)

| File | Perubahan |
|------|-----------|
| `CMakeLists.txt` | `MOCKTAIL_NATIVE_OPTIMIZATION` default **ON** (`-march=native -mtune=native -fno-plt` untuk Excavator); `-fomit-frame-pointer` pada Release; `-O2` pada Release (turun dari `-O3` untuk stabilitas); `-ffunction-sections -fdata-sections` + `-Wl,--gc-sections` (hapus kode unused); `-Wl,--strip-all`; `-DNDEBUG`; `MOCKTAIL_FAST_MATH=OFF` (dimatikan karena menyebabkan crash); `CPACK_STRIP_FILES TRUE`; opsi modular: `MOCKTAIL_ENABLE_AUDIO`, `MOCKTAIL_ENABLE_UPDATE`, `MOCKTAIL_ENABLE_WEBVIEW`, `MOCKTAIL_ENABLE_DISCORD_RPC` (semua default ON) |
| `Makefile` | `JOBS ?= 2` (hindari OOM saat link pada CPU 2-core, RAM 7.2 GiB) |

### 2. Compat Layer (`src/compat/`)

| File | Perubahan |
|------|-----------|
| `bionic_host_libc_runtime.cc` | `BionicMallinfoHasHostTelemetry()` selalu return `false` (telemetry dimatikan) |
| `bionic_abi_exports.cc` | (direvert ke asli — `__builtin_expect` menyebabkan segfault di `nativeSetAssetPath`) |
| `host_allocator_bridge.cc` | (direvert ke asli — 8-shard mutex + TLS cache menyebabkan segfault) |

### 3. Libc Shim (`src/libc_shim/`)

| File | Perubahan |
|------|-----------|
| `libc_shim.cc` | (direvert ke asli — TLS path cache menyebabkan segfault) |
| `vulkan_etc1_sky_transcoder.cc` | (direvert ke asli — SSE4.2 SIMD menyebabkan segfault) |

### 4. Linker (`src/linker/`)

| File | Perubahan |
|------|-----------|
| `linker.cc` | (direvert ke asli — TLS symbol cache menyebabkan segfault) |

### 5. JNI (`src/jnivm/`)

| File | Perubahan |
|------|-----------|
| `jni_references.cc` | (direvert ke asli — thread-local reverse map menyebabkan segfault) |
| `jnivm.cc` | (direvert ke asli — DisplaySizeCache menyebabkan segfault) |

### 6. Audio (`src/audio/`)

| File | Perubahan |
|------|-----------|
| `opensl_simple_buffer_queue.cc` | (direvert ke asli — stack size 256KB→128KB menyebabkan segfault) |

### 7. Runtime (`src/runtime/`)

| File | Perubahan |
|------|-----------|
| `graphics_launch_policy.cc` | Default backend **OpenGL ES** (`kSystem`) untuk Radeon R3; present mode `fifo` (vsync on); **MSAA off**; texture quality **low**; render scale **0.75** (semua backend, bukan hanya kSystem) |
| `frame_rate_policy.cc` | Default frame rate `"display"` (ikuti refresh layar 60 Hz), bukan unlimited |
| `device_memory_profile.cc` | Host <8 GiB RAM: `memory_class_mb` 512→384, `large_memory_class_mb` 1024→768 |
| `memory_limit.cc` | `ReadCgroupV2MemoryMax()` baca `/sys/fs/cgroup/memory.max`; cgroup limit reexec opsional |
| `texture_memory_policy.cc` | `kMaximumBudgetBytes` 1536 MiB → **512 MiB** (hemat RAM untuk iGPU shared) |
| `roblox_message_bus_request_bridge.cc` | Hapus LocalString telemetry; kirim `nullptr` ke `publish_response` (telemetry dimatikan) |
| `crash_report_policy.cc` | Blokir crash report ke host & simbol crashpad (sudah ada) |
| `bionic_dns_runtime.cc` | Blokir domain crash report (crashes.rbxinfra.com, backtrace.rbx.com, backtrace.io) (sudah ada) |

### 8. Graphics (`src/graphics/`)

| File | Perubahan |
|------|-----------|
| `etc2_decoder.cc` | `__builtin_expect` pada rare branches; loop unroll pada 16-texel paint block |
| `graphics_launch_policy.cc` | (lihat di atas) |
| `present_mode_policy.cc` | `ANV_SYS_MEM_LIMIT` + `MESA_VK_ENABLE_SUBMIT_THREAD` hanya untuk Intel GPU |

### 9. Config (`config/`)

| File | Perubahan |
|------|-----------|
| `roblox_compatibility.json` | `client_settings` FFlag grafis ringan ke 5 profil: `FIntRenderForceVideoMemorySize=128`, `FIntRenderTextureTotalBudgetMB=64`, `FIntAvatarTextureMemoryMax=16777216`, `FFlagRenderEnableLowEndLOD=True`, `FFlagDebugTerrainVTCompressedTextures=True`, `FFlagStreamingObserverMemoryOptimization=True` |

### 10. Optimasi yang Direvert (menyebabkan segfault)

Berikut optimasi yang **dicoba tapi direvert** karena menyebabkan crash (segfault) saat `nativeSetAssetPath`:

| File | Perubahan yang dicoba | Alasan direvert |
|------|----------------------|-----------------|
| `src/compat/host_allocator_bridge.cc` | 8-shard mutex, TLS allocation cache (kTlsCacheSize=4), padding kecil, malloc bukan calloc | Segfault — reuse freed allocation可能 menyebabkan use-after-free |
| `src/linker/linker.cc` | TLS symbol resolution cache (kTlsSymbolCacheSize=16), shared_mutex, reverse map handle→nama | Segfault — dangling pointer saat library di-unload |
| `src/libc_shim/libc_shim.cc` | TLS path lookup cache (kTlsPathCacheSize=8), sorted vector TranslatePath | Segfault — stale path cache |
| `src/compat/bionic_abi_exports.cc` | `__builtin_expect` fast-path pada mutex/once/spin | Segfault |
| `src/jnivm/jni_references.cc` | Thread-local reverse map untuk O(1) UnregisterLocalRef | Segfault — iterator invalidation |
| `src/jnivm/jnivm.cc` | DisplaySizeCache, `__builtin_expect` pada IntResultForName | Segfault |
| `src/audio/opensl_simple_buffer_queue.cc` | Stack size 256KB → 128KB | Segfault |
| `src/libc_shim/vulkan_etc1_sky_transcoder.cc` | SSE4.2 SIMD FindBestPaletteIndexSse42 | Segfault |

## Hasil Verifikasi

Binary berhasil di-build dan di-test:

- **Ukuran binary:** 3.5 MB (stripped)
- **OpenGL backend:** exit 124 (timeout, `exit_status=0`) — tidak ada crash
- **direct-vulkan backend:** exit 124 (timeout, `exit_status=0`) — tidak ada crash
- **LTO=1:** masih broken (undefined reference stub+audio)

## Perintah Build & Jalankan

```bash
# Build
JOBS=1 NATIVE=1 LTO=0 make build

# Jalankan dengan OpenGL ES + vsync 60 Hz
MOCKTAIL_FRAME_RATE_LIMIT=display MOCKTAIL_VSYNC=on ./build/mocktail --graphics opengl

# Atau dengan Vulkan
./build/mocktail --graphics direct-vulkan
```

## Catatan

- `JOBS=1` lebih stabil daripada `JOBS=2` pada CPU 2-core (hindari OOM saat linking).
- Semua optimasi memori agresif (TLS cache, sharded allocator, SIMD transcoder) direvert karena menyebabkan segfault pada engine Roblox. Optimasi yang tersisa (build flags, graphics policy, memory budget, telemetry disable) aman dan terverifikasi.