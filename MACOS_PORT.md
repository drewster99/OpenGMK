# macOS Port - Research & Plan

## Overview

This document summarizes research into porting OpenGMK to macOS, including the current state of the codebase, the GM8.2 extension ecosystem, and what work is required.

## Current Platform Support

| Platform | Windowing | OpenGL Context | DLL/Extensions | Status |
|---|---|---|---|---|
| Windows x86 | ramen (Win32) | WGL | Native loading | Full support |
| Windows x64 | ramen (Win32) | WGL | WoW64 IPC bridge | Full support |
| Linux (X11) | ramen (X11) | GLX | Dummy stubs | Works (no DLLs) |
| macOS | **None** | **None** | Dummy stubs | **Not supported** |

## What Needs to Be Done

### 1. Windowing Backend (BLOCKING)

The `ramen` crate (custom fork at `viriuwu/ramen`) only has Win32 and Linux/X11 backends. No macOS/Cocoa/AppKit backend exists.

**Options (pick one):**
- **Write a macOS backend for ramen** — Cocoa/AppKit via the `objc` crate. ~300-500 lines based on WGL (292 lines) and GLX (416 lines) backend sizes.
- **Replace ramen with `winit`** — mature, well-maintained cross-platform windowing. More refactoring upfront but eliminates the platform gap permanently.
- **Replace ramen with SDL2** — proven cross-platform. `gm82joy` already ships SDL2, so there's precedent in the GM8.2 ecosystem.

Replacing ramen entirely is likely the better long-term choice since it's a niche custom library with no macOS support and limited maintenance.

### 2. OpenGL Context Creation (BLOCKING)

Currently: `wgl.rs` (292 lines, Windows) and `glx.rs` (416 lines, Linux/X11).

Need a CGL (Core OpenGL) or NSOpenGLView backend for macOS. ~400 lines estimated.

**Important caveat:** macOS deprecated OpenGL (since 10.14 Mojave) but still supports up to OpenGL 4.1. OpenGMK requires OpenGL 3.3, so it works on current macOS but could break in a future release.

**Longer-term rendering options:**
- **ANGLE** (Google's OpenGL-to-Metal translator) — GM8.2 itself uses this via `gm82angle`, so there's precedent
- **wgpu** — Rust's cross-platform GPU abstraction (backends for Metal, Vulkan, DX12, OpenGL)
- **MoltenGL** — commercial OpenGL-to-Metal bridge
- Rewrite the renderer to target **Metal** directly

For an initial port, CGL + OpenGL 3.3 is the fastest path and works fine today.

### 3. Platform Kernel Functions (NON-BLOCKING)

The `#[cfg(target_os = "windows")]` blocks in `kernel.rs` and `game.rs` (~10 spots). On Linux these already fall back to hardcoded values, so the same would work on macOS. Proper implementations are trivial:

| Function | macOS Implementation |
|---|---|
| `display_get_width` | `CGDisplayPixelsWide()` |
| `display_get_height` | `CGDisplayPixelsHigh()` |
| `display_get_colordepth` | `CGDisplayBitsPerPixel()` |
| `display_get_frequency` | `CGDisplayCopyDisplayMode()` |
| `disk_free` / `disk_size` | `statvfs()` (POSIX, same as Linux) |
| `window_handle` | Return NSWindow pointer |

### 4. Audio (NEEDS INVESTIGATION)

Audio uses `rmp3` (pure Rust MP3 decoder) and `udon` (WAV). No obvious platform-specific audio output code in the emulator — likely handled through ramen or another dependency. Need to verify audio output actually works on macOS or if a platform-specific audio backend (CoreAudio) is needed.

### 5. External DLL System (SAME AS LINUX)

The `dummy.rs` fallback is used for any non-Windows-x86 target. macOS gets the same dummy stubs as Linux — `external_define` succeeds silently but `external_call` panics. This means games using external DLLs won't work, same as Linux.

## GM8.2 Extension Ecosystem

Every GM8.2 game depends on `gm82core` at minimum. The other extensions are optional. All are fully open-source with C/C++ source available.

### gm82core (REQUIRED for any GM8.2 game)
- **351 total functions**: 128 native DLL + 223 GML scripts
- **Repository:** [GM82Project/gm82core](https://github.com/GM82Project/gm82core)
- **Source files:** `gm82core.c`, `math.c`, `perlin.c`, `lovey01.c`, `hrt.c`, `windows.c`, `terrible_gm8_hacking.c`
- **Portability breakdown:**
  - `math.c` — pure math (angle, lerp, trig, geometry). Trivially portable, no OS APIs.
  - `perlin.c` — Perlin noise generation. Portable.
  - `lovey01.c` — additional math/geometry. Portable.
  - `gm82core.c` — string tokenizer, color utilities. Portable.
  - `hrt.c` — High Resolution Timer. Uses Windows `QueryPerformanceCounter`. macOS equivalent: `mach_absolute_time()`.
  - `windows.c` — **Hard part.** Window handle manipulation, WndProc subclassing, registry access, process management. Deeply Win32-specific, needs full reimplementation.
  - `terrible_gm8_hacking.c` — patches the GM8 runner in memory. Irrelevant for OpenGMK (we *are* the runner).
- **OpenGMK dll-emulation-2 coverage:** 1 of 128 DLL functions (`resize_backbuffer`). **0.3%**

### gm82dx9 (DirectX 9 rendering)
- **257 total functions**: 146 native DLL + 111 GML scripts
- **Repository:** [GM82Project/gm82dx9](https://github.com/GM82Project/gm82dx9)
- **Source files:** `gm82dx9.cpp`, `inject.cpp`, `transform.cpp`, `vertex_buffers.cpp`, `shaders.cpp`, HLSL shaders
- All Direct3D 9 specific. Would need complete rewrite against OpenGL/Metal/wgpu.
- Provides: HLSL shaders, vertex buffers, vertex formats, safe surfaces, separate alpha blending, backbuffer resize, scissor test, improved lighting
- **OpenGMK coverage:** 0%

### gm82snd (audio engine replacement)
- **256 total functions**: all GML wrappers around FMOD Ex
- **Repository:** [GM82Project/gm82snd](https://github.com/GM82Project/gm82snd)
- Source is all GML scripts (`fmod_compat.gml`, `newsound.gml`, `shims.gml`, etc.) that call into `fmodex.dll` (closed-source FMOD Ex library)
- Supports: WAV, OGG, MP3, MIDI, MOD, IT, S3M, XM, AIF, FLAC, WMA, streaming, 3D positional audio, effects chains
- **Cross-platform path:** Replace FMOD Ex calls with a portable audio library (miniaudio, SDL_mixer, or modern FMOD Core)
- **OpenGMK coverage:** 0%

### gm82buf (memory buffers, sockets, hashing)
- **113 total functions**: 109 native DLL + 4 GML scripts
- **Repository:** [GM82Project/gm82buf](https://github.com/GM82Project/gm82buf)
- **Source files:** `Buffer.cpp`, `Socket.cpp`, `Hash.cpp`, `gm_pipe.cpp`, `gm_udpsocket.cpp`
- Modified version of Maarten Baert's HTTPDLL2
- Sockets use Winsock — maps directly to BSD sockets on macOS/Linux
- **OpenGMK coverage:** 0%

### gm82joy (joystick/gamepad input)
- **37 total functions**: all GML wrappers
- **Repository:** [GM82Project/gm82joy](https://github.com/GM82Project/gm82joy)
- Single C file (`gm82joy.c`) that wraps SDL2
- Ships `SDL2.dll` — SDL2 is already cross-platform, so this is the easiest extension to port
- **OpenGMK coverage:** 0%

## DLL Emulation Framework

The `dll-emulation-2` branch (last touched October 2021) established a pattern for native reimplementation of DLL functions:

1. When `external_define` is called, read the PE timestamp from the DLL file header
2. Look up known timestamps in a compile-time `phf_map` to find emulated function tables
3. Each emulated DLL maps function names to Rust implementations via `phf_map`
4. Dispatch priority: dummy audio stubs -> emulated functions -> native Win32 loading -> IPC bridge

The framework exists but almost no functions are actually implemented.

### surface_fix (pre-8.2 community extension)
- 9 functions total
- **3 working:** `FixSurfaces`, `GetCurrentSurface`, (+ `fix_surfaces` toggle)
- **6 `unimplemented!()`:** `ClearDepthBuffer`, `SurfaceToString`, `SurfaceFromString`, `WriteSurfaceToBinaryFile`, `ReadSurfaceFromBinaryFile`, `ChangeDepthBuffer`, `EnableDepthWriting`

## Related Projects

- **[Dejavu](https://github.com/rpjohnst/dejavu)** — alternative GM8 runner in Rust with a register-based bytecode VM. Has a working WASM web playground. Uses WebGL 2 and Direct3D 11. Less complete than OpenGMK but architecturally cleaner.
- **[GM82Project/gm82save](https://github.com/GM82Project/gm82save)** — the .gm82 project format parser (Rust, ~2700 lines). The format is a directory of plain text files and PNGs, trivially parseable.
- **[OpenGMK issue #134](https://github.com/OpenGMK/OpenGMK/issues/134)** — WASM/RetroArch port request (no progress)
- **[OpenGMK issue #101](https://github.com/OpenGMK/OpenGMK/issues/101)** — bundle engine + game as single executable (discussed, `opengmk_as_runner` branch has initial work)

## Branch Status in OpenGMK

| Branch | Purpose | Last commit | Status |
|---|---|---|---|
| `master` | Main development (TAS features, compat fixes) | Dec 2025 | Active |
| `dll-emulation` | Major refactor + DLL emulation framework | April 2021 | Abandoned |
| `dll-emulation-2` | Cleaner DLL emulation + gm82 room data | October 2021 | Abandoned |
| `opengmk_as_runner` | Self-reading executable (game data appended to binary) | September 2021 | Abandoned |

## Estimated Effort Summary

| Work item | Effort | Blocking for macOS? |
|---|---|---|
| Windowing backend (ramen macOS or replace with winit/SDL2) | Medium | **Yes** |
| CGL/NSOpenGL context creation | Small (~400 lines) | **Yes** |
| Platform kernel functions (display, disk info) | Trivial (~50 lines) | No (fallbacks exist) |
| Audio output verification/fix | Unknown | Maybe |
| gm82core math/string/color functions | Small (trivially portable C) | No (DLL emulation) |
| gm82core window/registry/system functions | Medium (Win32-specific) | No (DLL emulation) |
| gm82dx9 (DirectX 9 -> OpenGL/Metal) | Large (full rewrite) | No (DLL emulation) |
| gm82snd (FMOD -> portable audio) | Large | No (DLL emulation) |
| gm82buf (Winsock -> BSD sockets) | Small-Medium | No (DLL emulation) |
| gm82joy (already SDL2-based) | Small | No (DLL emulation) |
