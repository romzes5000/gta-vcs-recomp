# macOS (Apple Silicon)

Status (2026-11-01, branch `macos-port`): the recompiler, the runtime, the Vulkan renderer (via MoltenVK) and the
game's own code compile and link natively for arm64 with Xcode's clang. Checked on an M4 with macOS 27. **Not yet run
with a real disc** - the first run with the ISO is the real test.

How it works on a Mac:
- x86 SSE in the runtime and in the recompiled code is translated to NEON by [sse2neon](https://github.com/DLTcollab/sse2neon)
  (fetched automatically). AVX paths are compiled out; the scalar fallbacks are used.
- Vulkan 1.3 through MoltenVK (Homebrew `molten-vk` + `vulkan-loader`). The renderer enables
  `VK_KHR_portability_enumeration` / `VK_KHR_portability_subset`. On an M4, MoltenVK reports Vulkan 1.3 with dynamic
  rendering, synchronization2, push descriptors, dual-source blending, depth clamp and clip distances - all the game needs.
- Window / input / audio: SDL3, built from source (Cocoa, CoreAudio, GameController - nothing to install).
- Linker: Apple's `ld` (ThinLTO with its own cache flags); `lld` is not needed.
- `PS2X_CPU` (AVX2 / AVX / SSE4) is ignored on arm64; the build uses `-mcpu=apple-m1` (runs on every M-series chip).

## 1. Tools (once)

```sh
xcode-select --install          # Apple clang 17+ (skip if Xcode is installed)
brew install cmake ninja pkgconf git glslang ffmpeg molten-vk vulkan-loader vulkan-headers freetype python
```

Check: `python3 tools/build.py --check-deps` (it prints the `brew install` line for anything missing;
`python3 tools/build.py --install-deps` runs it). Homebrew LLVM (`brew install llvm`) is optional - used when present.

## 2. The disc

Put your own **Grand Theft Auto: Vice City Stories (USA, SLUS-21590)** PS2 image in the repository root, named so it
matches `*Vice City Stories*.iso`, e.g.

```sh
cd ~/games/vcs-recomp
cp "/path/to/Grand Theft Auto - Vice City Stories (USA).iso" .
```

(or pass the path to the commands below, or `export VCS_ISO=/path/to/file.iso`). The PSP version does not work.

## 3. Recompile and build

```sh
python3 tools/build.py --recompile
```

The first run downloads the dependencies (CMake FetchContent, a few minutes), recompiles the EE code from the disc into
`generated/`, then builds the game with ThinLTO (several minutes on an M-series Mac). `--quick` skips LTO for faster
rebuilds.

## 4. Play

```sh
python3 tools/run.py 0          # runs until you quit, log in build/run/run.log
# or directly:
build/vcs                       # finds the .iso next to the repo / via vcs.ini / first argument
```

## Known gaps / risks

- Untested end to end without a disc. Expect rendering issues to be MoltenVK translation limits (look at
  `build/run/run.log`; `MVK_CONFIG_LOG_LEVEL=2` shows MoltenVK's warnings).
- Performance: SSE->NEON is mostly 1:1, but the AVX-specialised VU1 paths fall back to scalar.
- The crash handler / sampling profiler (`src/debug/crash.cpp`, `profiler.cpp`) are Linux/Windows-only: on macOS a
  crash gives no PS2-side report.
- Write-watch debugging (userfaultfd) is Linux-only; it is a debug feature.
- The launcher (`--launcher`) was not ported.
- The `PS2_GUEST_CC` calling convention (`preserve_none`) is x86-64-only here; arm64 uses the default convention
  (correct, a bit slower). Enabling it on AArch64 needs clang 19+ (`brew install llvm`).
