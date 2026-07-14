# chibi-recompo

A native static-recompilation project for the US GameCube release of
**Chibi-Robo! Plug Into Adventure!** (`GGTE01`). It is built on:

- [DolRecomp](https://github.com/ExpansionPak/DolRecomp) - converts the game's DOL into split C source.
- [ModernGekko](https://github.com/xXJSONDeruloXx/ModernGekko/tree/chibi-robo-native) - runtime and tooling for the native module.
- [RecompCore](https://github.com/xXJSONDeruloXx/RecompCore/tree/chibi-robo-native) - Dolphin-derived chassis with the Chibi-Robo correctness and profiling fixes.

The dependencies are pinned as submodules under `lib/`, and the top-level
`Makefile` drives extraction, recompilation, module compilation, and launch.
No game data is included.

## Current Status

- The native module boots through the health warning and logos into gameplay.
- X11/Vulkan rendering and keyboard controls are working.
- `blrl`/`bclrl`, asynchronous interrupt delivery, and native/interpreter
  fallback handoff are fixed and differentially validated.
- GGTE01 uses dual-core execution and three measured idle-loop PCs. Bounded
  native backedges batch copy/clear loops while yielding every 256 guest cycles.
  Together these reduced steady 3D dispatch pressure from roughly 53 million to
  4–6 million dispatches per second.
- CPU ABI v3 supports Dolphin-compatible fast FP defaults while retaining exact
  FP behavior for lockstep verification, `AccurateNaNs`, and enabled guest FP
  exceptions.
- Native gameplay usually sustains `59.94 VPS / 100%`, but a recurring 3D
  workload still drops to roughly `40–46 VPS / 66–79%` for several seconds.
  JIT64 sustains 100% on the same path and remains comparison-only, not the
  deliverable.
- Rolling profiling identifies the remaining spike as call/return-heavy native
  dispatch around `0x801ADD44`, `0x801ACFC4`, and the copy routine at
  `0x8000519C`; cache fallback traffic does not spike with it.

## Dependencies

CMake, Ninja, and pkg-config, plus a C11/C++23 toolchain. DolRecomp and ModernGekko both build on macOS, Linux, and Windows; this repo's `Makefile` doesn't do anything platform-specific itself, it just drives the same CMake builds each submodule supports natively.

On macOS, via Homebrew:

```
brew install cmake ninja pkg-config
```

Xcode's command line tools are also required (AppleClang 14.0.3+; verified on AppleClang 17).

## Getting the Source

```
git clone --recurse-submodules https://github.com/xXJSONDeruloXx/chibi-recompo.git
cd chibi-recompo
```

If you already cloned without `--recurse-submodules`, `make` will fetch them for you on first run. To do it manually instead:

```
git submodule update --init --recursive
```

> [!NOTE]
> `lib/ModernGekko` vendors a large chunk of Dolphin's dependency tree (SDL, fmt, imgui, Vulkan headers, etc.), so the first submodule sync takes a while and pulls a few hundred MB.

## Recompile and Run

Bring your own legally-owned `GGTE01` ISO; no game data is included in or downloaded by this repository. There is no bundled default image, so `ISO=` (or an already-extracted `GAME=`) is required. Point `ISO` at your dump and run:

```
make run ISO=/path/to/Chibi-Robo.iso RUN_ARGS="--graphics Vulkan --audio ALSA -X11"
```

This builds DolRecomp and ModernGekko, extracts the ISO, recompiles `main.dol` to C, compiles the result into a native module, and launches the game in a window.

Each game gets its own directory under `extracted/<slug>/`, where `<slug>` is derived from the ISO's filename, so multiple games coexist without clobbering each other. `ISO` is only needed the first time per game, once extracted, run it again by slug instead:

```
make run ISO=iso/Your\ Game.iso        # first time for a new game
make run GAME=Your-Game-Slug           # afterwards
```

Drop ISOs under `iso/` at the repo root (gitignored) for a stable local path.

To just build the tools without touching a game, or to produce the compiled module without launching it:

```
make tools
make recompile ISO=/path/to/game.iso
```

`moderngekko-port` caches compiled modules by DOL hash, toolchain identity,
DolRecomp executable hash, and the module runtime-source hash. Re-running
`recompile`/`run` after the first build is therefore cheap without risking a
stale module after emitter or GXRuntime changes.

> [!NOTE]
> Wii ISOs need [Wiimms ISO Tools](https://wit.wiimm.de/) (`wit`) for extraction, `make` downloads it automatically into `extern/wit` on first use. GameCube extraction is built into DolRecomp directly and doesn't need this.

## Makefile Targets

Run `make help` (or just `make`, the default target) for this list:

| Target       | Description                                                |
|--------------|--------------------------------------------------------------|
| `tools`      | Build DolRecomp and ModernGekko                             |
| `extract`    | Extract a GameCube/Wii ISO into `extracted/<slug>/`         |
| `recompile`  | Recompile + compile a runnable module                       |
| `run`        | Recompile (if needed) and launch the game                   |
| `clean`      | Remove all build output                                     |

## Variables

| Variable           | Default                                    | Description                                              |
|--------------------|--------------------------------------------|----------------------------------------------------------|
| `ISO`              | *(none)*                                   | Path to a game ISO. Required the first time per game, also determines that game's slug. |
| `GAME`             | *(none)*                                   | Select an already-extracted game by slug instead of `ISO=`. Required if `ISO` isn't given. |
| `JOBS`             | detected CPU count                         | Parallel build jobs passed to CMake/Ninja.                |
| `CMAKE_BUILD_TYPE` | `Release`                                  | Passed to both submodule builds.                          |
| `RUN_ARGS`         | *(empty)*                                  | Extra flags forwarded to `moderngekko-run` via `make run`, e.g. `--headless`, `--graphics Vulkan`. |

For example, to force a debug build with extra runner flags:

```
CMAKE_BUILD_TYPE=Debug make run ISO=/path/to/game.iso RUN_ARGS="--headless"
```

## Native/JIT Comparison and Profiling

The runner defaults to the native module. Select a core explicitly for a
reproducible comparison:

```
make run GAME=chibi-robo RUN_ARGS="--cpu-core static --graphics Vulkan -X11"
make run GAME=chibi-robo RUN_ARGS="--cpu-core jit64 --graphics Vulkan -X11"
```

Optional one-second performance telemetry and sampled native-dispatch timing
are enabled with environment variables:

```
MODERNGEKKO_PROFILE_PERF=1 STATICRECOMP_PROFILE_DISPATCH=1 \
  make run GAME=chibi-robo RUN_ARGS="--cpu-core static --graphics Vulkan -X11"
```

## Controller Input

ModernGekko has no in-app controller configuration UI (same as Dolphin's NoGUI frontend it's built on), bindings come from `Config/GCPadNew.ini` / `Config/WiimoteNew.ini` in ModernGekko's user directory (`~/.local/share/moderngekko/Config/` by default), which nothing in this pipeline creates for you. Without one, GameCube pad input silently does nothing. You'll need to hand-author or copy in a working ini, Dolphin's ini format and key names are stable and documented by the project itself.

## Cleaning Up

```
make clean           # everything below
make clean-extracted # extracted disc + compiled modules only, keeps built tools
make clean-tools     # DolRecomp/ModernGekko build trees only
```

## License

DolRecomp and ModernGekko are each distributed under their own upstream licenses (see `lib/DolRecomp/LICENSE` and `lib/ModernGekko/LICENSE`, the latter GPL-3.0-or-later due to its Dolphin-derived runtime). No Nintendo disc image, extracted game data, keys, or copyrighted assets are part of this repository, bring your own legally-owned dump.
