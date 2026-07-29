# whisper.cpp on QNX 8.0 — Developer Handoff

## Overview

This fork ports whisper.cpp (speech-to-text, built on `ggml`) to QNX 8.0. It reuses
the QNX build fixes already proven on this same box for
[llama.cpp](https://github.com/qnx-ports/llama.cpp) and
[stable-diffusion.cpp](https://github.com/qnx-ports/stable-diffusion.cpp), which are
also `ggml`-based. Unlike those two, whisper.cpp vendors `ggml` inline rather than as
a submodule, so all patches below live directly in this repo's `ggml/` subtree.

CPU-only inference is verified end-to-end on both an x86_64 QEMU dev box and real
ARM64 hardware (Raspberry Pi 5). The Vulkan backend builds and correctly enumerates
the GPU device on x86_64/QEMU, but real GPU inference dispatch has **not** been
runtime-tested there — see "Vulkan status" below before relying on it. Vulkan was
not evaluated on ARM at all this round (CPU-only was the ask).

---

## Environment (this dev box)

| Component | Details |
|-----------|---------|
| OS | QNX 8.0.0 (x86_64, QEMU VM) |
| GPU | Virtio-GPU Venus (Intel UHD Graphics 620) via QEMU, 17.8 GiB shared memory |
| Compiler | clang/clang++ 21.1.3 |
| Build | CMake + Ninja |

`CMAKE_SYSTEM_PROCESSOR` reports `unknown` on this box, so `ggml-cpu` falls back to
`GGML_CPU_GENERIC` (no AVX/SSE dispatch). Harmless, just no CPU-specific
vectorization — same behavior seen porting llama.cpp and stable-diffusion.cpp here.

---

## QNX patches applied (branch `qnx-8.0`)

All four are adapted from the same fixes already applied in
[`qnx-ports/ggml`](https://github.com/qnx-ports/ggml) for the llama.cpp/sd.cpp ports:

1. **`ggml/src/ggml-vulkan/CMakeLists.txt`** — disable cooperative-matrix shader
   generation on QNX (`if(NOT QNXNTO)`). glslc's basic coopmat feature test passes on
   QNX, but the full shader variants fail intermittently to generate under QNX's
   resource constraints, and the gfxstream/Venus ICD doesn't expose cooperative
   matrix support at runtime anyway. This whisper.cpp ggml snapshot has a third
   coopmat-family test (`GL_NV_cooperative_matrix_decode_vector`) not present in the
   older snapshot vendored by sd.cpp/llama.cpp — guarded it too, same rationale.
2. **`ggml/src/ggml-vulkan/vulkan-shaders/vulkan-shaders-gen.cpp`** —
   `acquire_compile_slot()` capped to `N=1` (fully serial glslc invocations) under
   `#if defined(__QNX__)`. QNX's default per-process FD limit (1000, set via
   `procnto -f`) causes concurrent glslc child processes to silently fail to write
   their `.spv` output under load. Removable once the target system image is built
   with a higher `-f` value (e.g. `procnto-smp-instr -f 32768`+) and that's been
   validated to eliminate the failures. **This makes Vulkan shader generation slow**
   (tens of minutes on this VM) — budget for it.
3. **`ggml/src/ggml-vulkan/ggml-vulkan.cpp`** — guard `vk::LayerSettingEXT` /
   `vk::LayerSettingsCreateInfoEXT` usage behind `#if VK_HEADER_VERSION >= 301`,
   falling back to constructing `vk::InstanceCreateInfo` without layer settings on
   older headers. QNX 8.0 ships Vulkan headers older than version 301. Layer
   settings are a debug/validation-only feature; the fallback doesn't affect
   inference correctness.
4. **`examples/server/CMakeLists.txt`** — link `socket` explicitly on QNX
   (`if(CMAKE_SYSTEM_NAME STREQUAL "QNX")`). QNX doesn't provide socket symbols via
   libc; `whisper-server` fails to link without this (`undefined reference to
   'recv@@libsocket.so.4'`).

5. **`ggml/cmake/common.cmake`** — `ggml_get_system_arch()`'s ARM-detection regex
   (`^(aarch64|arm.*|ARM64)$`) requires an exact match against
   `CMAKE_SYSTEM_PROCESSOR`, but QNX reports `aarch64le` (endianness-qualified, not
   bare `aarch64`) on ARM64 targets. This silently fell through to
   `GGML_SYSTEM_ARCH=UNKNOWN`, and `ggml-cpu` built with `GGML_CPU_GENERIC` — no
   NEON/dotprod/FMA codegen at all, correctness unaffected but a real perf loss.
   Widened the regex to `^(aarch64(le|be)?|arm.*|ARM64)$`. Verified on a Raspberry
   Pi 5: before the fix, generic-only; after, `GGML_SYSTEM_ARCH` resolves to `ARM`,
   `HAVE_DOTPROD`/`HAVE_FMA` checks pass, and `whisper-cli` reports `NEON = 1 |
   ARM_FMA = 1 | DOTPROD = 1` at runtime. This is an ARM-only fix and doesn't affect
   the x86_64 build (which has its own separate, unrelated `CMAKE_SYSTEM_PROCESSOR:
   unknown` quirk on this QEMU box — see the CPU-only build note below). **Likely
   applicable to llama.cpp/stable-diffusion.cpp too** if either is ever built on
   ARM64 QNX — their vendored copies of this same file have the identical unfixed
   regex, unexercised so far because both were only built on x86_64 QEMU here.

**Checked, not needed:** the `--whole-archive` linker workaround that sd.cpp's
top-level `SD_LIB` target required (its SPIR-V shader objects get silently dropped
by the linker when statically linking `ggml-vulkan` into a shared lib, because
they self-register via global constructors with no externally-referenced symbol).
whisper.cpp only links the `ggml` umbrella target — same pattern as llama.cpp, which
never hit this — so no equivalent explicit static link of `ggml-vulkan` exists here.
Confirmed by a clean `GGML_VULKAN=ON` build with no missing-symbol issues. If real
GPU inference ever surfaces a "missing op" symptom, re-check this.

---

## Build instructions

CPU-only:

```sh
cmake -B build_cpu -G Ninja \
  -DCMAKE_C_COMPILER=/usr/bin/clang -DCMAKE_CXX_COMPILER=/usr/bin/clang++ \
  -DCMAKE_BUILD_TYPE=Release -DGGML_VULKAN=OFF -DWHISPER_SDL2=OFF
cmake --build build_cpu -j$(nproc)
```

Vulkan (adds the coopmat-disabled, serially-generated shaders above — expect this
to take much longer than the CPU build):

```sh
cmake -B build_vulkan -G Ninja \
  -DCMAKE_C_COMPILER=/usr/bin/clang -DCMAKE_CXX_COMPILER=/usr/bin/clang++ \
  -DCMAKE_BUILD_TYPE=Release -DGGML_VULKAN=ON -DWHISPER_SDL2=OFF
cmake --build build_vulkan -j$(nproc)
```

`WHISPER_SDL2` is left off (default OFF) — the `stream`/`command`/`talk-llama`/`lsp`
examples that need it were not evaluated on QNX this round. Everything else that
builds by default (`whisper-cli`, `whisper-server`, `whisper-bench`,
`whisper-quantize`, `parakeet-cli`, `parakeet-quantize`,
`whisper-vad-speech-segments`) builds and links cleanly in both configs.

### CPU inference — verified working

```sh
sh ./models/download-ggml-model.sh tiny.en
./build_cpu/bin/whisper-cli -m models/ggml-tiny.en.bin -f samples/jfk.wav
```

Produces correct output: *"And so my fellow Americans ask not what your country can
do for you, ask what you can do for your country."*

---

## ARM64 (aarch64) — verified on QNX 8.0, Raspberry Pi 5

CPU-only build and inference verified end-to-end on real ARM hardware, not just
QEMU. Same CMake invocation as the x86_64 CPU-only build above, just with this
box's toolchain path (found via `which clang clang++`) and `-j4` (RPi 5, 4 cores):

```sh
cmake -B build_cpu -G Ninja \
  -DCMAKE_C_COMPILER=/system/bin/clang -DCMAKE_CXX_COMPILER=/system/bin/clang++ \
  -DCMAKE_BUILD_TYPE=Release -DGGML_VULKAN=OFF -DWHISPER_SDL2=OFF
cmake --build build_cpu -j4
```

The compiler path differs from the x86_64 QEMU dev box (`/system/bin/clang` here
vs. `/usr/bin/clang` there) — this appears to be a difference between QNX device
images rather than an architecture-specific requirement; check `which clang` on
whatever box you're building on rather than assuming either path.

No source patches were needed beyond the arch-detection fix in patch 5 above (which
is what makes this build actually use NEON instead of silently falling back to
generic). Confirmed via `whisper-cli`'s runtime `system_info` line:

```
system_info: n_threads = 4 / 4 | WHISPER : COREML = 0 | OPENVINO = 0 | CPU : NEON = 1 | ARM_FMA = 1 | DOTPROD = 1 | OPENMP = 1 | REPACK = 1 |
```

and correct transcription output from the same `tiny.en` + `samples/jfk.wav` test
used on x86_64. Encode time was roughly 2x faster than the x86_64 QEMU box's
generic-fallback build (3.0s vs. 6.0s) despite having half the cores (4 vs. 8) —
consistent with NEON dotprod genuinely being used rather than a fluke.

Vulkan was not attempted on this ARM board this round — CPU-only was the ask, and
whether this specific Pi's GPU/driver stack even exposes a usable Vulkan ICD under
QNX was never investigated. Treat that as a fully open question for whoever picks
it up next, not as "probably works like x86_64."

---

## Vulkan status — enumeration verified, real inference dispatch NOT tested here

**What's confirmed:** the Vulkan backend builds, links (`whisper-cli`,
`whisper-server`, and everything else), and correctly enumerates the GPU:

```
ggml_vulkan: Found 1 Vulkan devices:
ggml_vulkan: 0 = Virtio-GPU Venus (Intel(R) UHD Graphics 620 (WHL GT2)) (venus) | uma: 1 | fp16: 1 | bf16: 0 | ...
```

Unlike `llama-cli`, `whisper-cli` has no built-in `--list-devices` flag, so this was
confirmed with a small standalone probe that calls the same underlying
`ggml_backend_dev_count()`/`ggml_backend_dev_get()`/`ggml_backend_dev_name()` API
`llama-cli --list-devices` uses internally — no `VkDevice` creation, no shader
compilation, just physical-device enumeration.

**What's deliberately NOT tested here:** an actual transcription run with
`-ng`/`--no-gpu` unset (i.e. real GPU compute dispatch). This QEMU box has a
previously-documented deadlock where a cold/uninitialized shader cache under the
`drm-virtio` resource manager can hang the GPU command path badly enough to require
a full VM reboot (root-caused during the stable-diffusion.cpp port — see that
project's notes). Given this round's goal was CPU-only functional verification, GPU
dispatch was intentionally left untested to avoid that risk.

### For whoever runs this on bare metal

1. **You need Screen compositor running before any Vulkan call**, same as the
   llama.cpp port found. On QNX, Vulkan appears to route through the Screen
   compositor generally (not just for this QEMU/Venus setup) — bare metal will
   likely still need Screen running, just pointed at a real vendor ICD instead of
   the Venus one. On this dev box:
   ```sh
   sudo /usr/bin/screen -c /usr/share/screen/graphics-virtio-virgl-virtual.conf &
   sudo chmod 666 /dev/dri/renderD128 /dev/dri/card0
   ```
   is the QEMU-specific config; bare metal will need whatever Screen graphics
   config matches your actual GPU.
2. **The `drm-virtio` deadlock is believed QEMU/Venus-specific** — it was
   root-caused to virtio-gpu's passthrough resource manager, which bare metal with a
   real vendor ICD never touches. This is an informed bet based on the diagnosis,
   not a guarantee; a real ICD could have its own bugs. Don't assume it away —
   validate real inference dispatch carefully the first time on real hardware, ideally
   with easy recovery (not a production box) in case something GPU-driver-specific
   still goes wrong.
3. **Shader cache**: the QNX-side `N=1` serialization fix above only affects
   *build-time* shader generation, not runtime shader compilation. Runtime shader
   compilation for Venus happens host-side (cached in the host's Mesa shader cache
   directory, e.g. via `MESA_SHADER_CACHE_DIR`) — irrelevant on bare metal with a
   native ICD, which will use its own driver-level cache.
4. Once you've validated real dispatch works, please fold that confirmation back
   into this doc / the upstream PR description.
