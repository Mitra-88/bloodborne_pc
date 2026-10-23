# AGENTS.md

Windows port of Bloodborne (`windows` is the default branch). The game's own x86-64 PS4 code
runs natively; only graphics are translated (to Vulkan). This is not an emulator.

Setup state for this machine (MSYS2 packages, submodules, configured `out/gpu`, test baseline) is
in [`docs/history/SESSION_2026-10-23.md`](docs/history/SESSION_2026-10-23.md).

## Layout, by what it owns

| Path | Owns |
|---|---|
| `src/` | C11 PS4 kernel/HLE runtime (`runtime_*.c`, `compat_win.c`) and `probe.c`, the ELF loader that enters guest code. |
| `gpu/` | C++23 Vulkan renderer derived from shadPS4 (`gpu/shadps4/`), online stack (`gpu/bbnet/`), in-game ImGui menu, FSR/DLSS/ML-FG. Built by CMake as `libbbgpu` (static on Windows). |
| `launcher/bbport_launcher_win.py` | Windows launcher (Tkinter). `bbport_launcher.py` is the Linux GTK one; do not edit it for Windows behavior. |
| `scripts/` | Python that parses `eboot.bin`, applies 1.09 patches, mods, saves, FOV, content, etc. |
| `run.py` | Windows counterpart of `run.sh`: prepares the game image, sets env vars, starts `bb-probe.exe`. |

## Build (Windows: MSYS2 CLANG64 shell, not PowerShell)

```bash
git submodule update --init --recursive   # none are initialized in a fresh clone
bash build.sh                             # -> out/bb-probe.exe, bb-gpu-capabilities.exe, bb-fg401-capabilities.exe, bb-play.exe
bash packaging/windows/package.sh         # -> dist/bbport-windows/ + zip; needs a python.org Python
```

- `build.sh` sources `msys2-env.sh` (PATH for `C:\msys64\clang64`), applies
  `gpu/patches/fsr-vulkan/*.patch` to the FSR-Vulkan submodule, then configures CMake/Ninja.
  A failed GPU build aborts; the full log is `out/gpu-build.log`.
- `build.sh` writes `out/gpu/compile_commands.json` (`CMAKE_EXPORT_COMPILE_COMMANDS`); the
  gitignored `.vscode/` points the C/C++ extension's IntelliSense at it, with the MSYS2 CLANG64
  clang as `compilerPath`, so editor flags and `third_party` includes match the build.
- Incremental work after the first build: `ninja -C out/gpu bb-probe`.
- `package.sh` looks for a python.org Python under `/c/Python3*` or
  `~/AppData/Local/Programs/Python3*`; if yours is elsewhere, pass `WINPYTHON=<path>`.
- `src/runtime*.c` is globbed by both `build.sh` and `gpu/CMakeLists.txt`; a new runtime file is
  picked up without editing build files. C code is `-Wall -Wextra -Werror`.
- `gpu/CMakeLists.txt` defines extra `EXCLUDE_FROM_ALL` test binaries:
  `ninja -C out/gpu <target>` then run `out/gpu/<target>` (e.g. `fsr4vk-post-test`,
  `command-buffer-test`, `motion-history-test`).

## Tests

Python suites run from the repo root; `tests/` is not a package, so `python -m unittest
tests.test_x` fails on imports. Use discovery:

```bash
python -m unittest discover -s tests -p "test_fov.py" -v     # one module
python -m unittest discover -s tests                          # all
```

- The suite is not green on a bare Windows checkout: launcher modules import Pillow (install it,
  or the `out/pyenv` from `package.sh`), and a few Linux-only tests (`test_run_settings.py` needs
  `bash`/`run.sh`, `test_packaged_vulkan.py`, symlink tests) fail. Judge your change by the
  modules you touched, not the total. The baseline and the exact red modules are recorded in
  [`docs/history/SESSION_2026-10-23.md`](docs/history/SESSION_2026-10-23.md).
- When driving tools from PowerShell, a native tool that writes to stderr (cmake warnings, `uv`
  progress) can make the wrapper report exit 1 even on success; trust the tool's own rc.
- C tests in `tests/*.c` are compiled and run only by `bash build.sh --test` on Linux. The Windows
  branch of `build.sh` exits before that, and CMake defines no targets for them, so Python tests
  that exec `out/runtime-test`, `out/bb-probe` (no `.exe`) skip here. There is no CI config in
  the repo.
- `tests/test_launcher_translations.py` is the gate for UI text: placeholders, status symbols and
  the main page texts must exist in all 13 languages. UI strings belong in `launcher/bbport_lang*.py`
  (English key, Russian inline, other languages in the tables), never hardcoded in the launcher.
  The in-game overlay (`gpu/shim/bbport_overlay.cpp`) is the opposite: its text is English only,
  so do not route it through the launcher translations.

## Conventions

- Local edits inside vendored shadPS4 code (`gpu/shadps4/`, `gpu/bbnet/vendor/shadps4/`) carry a
  `bbport:` comment. `gpu/bbnet/thd/` is a different origin (adapted from droogie/bbhost,
  GPL-3.0-or-later), not shadPS4; `gpu/bbnet/SHADNET-PORT.md` records both provenances.
- Do not commit edits inside `gpu/third_party/fsr-vulkan`: record them as a numbered patch in
  `gpu/patches/fsr-vulkan/` (see its README). That submodule showing as modified after a build is
  expected.
- `VERSION` lives in `launcher/bbport_launcher_win.py`; the launcher's own settings are in
  `%APPDATA%/bbport-launcher/`. Game settings (`bbport.ini`, `input.ini`) and `user/` are runtime
  state and gitignored; `out/`, `dist/`, `*.png` (except `launcher/` assets) are ignored too.

## Run

```bash
BB_GAME_DIR=F:/path/to/CUSA03173 python run.py     # needs a decrypted 1.09 dump
```

Without a game folder, verification stops at build + the tests above.

## Read before touching a subsystem

- `packaging/windows/README.md` — build packages, FSR4/DLSS setup, and the Linux-vs-Windows table
  (TEB TLS slot, pagefile-backed guest memory, vectored exception handler + `bb_setjmp`, committed
  guest stacks, winpthreads quirks). Read it before changing loader, memory or thread code.
- `docs/INPUT.md` (input config), `docs/FSR411_WINDOWS.md`, `docs/ML_FRAME_GENERATION_WINDOWS.md`,
  `docs/WINDOWS_TROUBLESHOOTING.md`, `docs/VRAM_PERFORMANCE.md` (where VRAM goes, `BB_GC_*` /
  `BB_VRAM_WARN_MB` / `BB_ARENA_IDLE_SECONDS`, what is left to optimize), `gpu/bbnet/SHADNET-PORT.md` (online),
  `docs/ROADMAP.md`.
