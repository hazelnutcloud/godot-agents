# GDExtension Build System Reference

A detailed description of the build process defined in `SConstruct` and `godot-cpp/`, intended to allow recreation with a different build tool (CMake, Meson, Ninja, etc.).

---

## Table of Contents

1. [Overview](#1-overview)
2. [File Map and Roles](#2-file-map-and-roles)
3. [Build Flow](#3-build-flow)
4. [Build Options / Variables](#4-build-options--variables)
5. [Platform Detection](#5-platform-detection)
6. [Architecture Detection](#6-architecture-detection)
7. [How `target` Affects the Build](#7-how-target-affects-the-build)
8. [Compiler Flags](#8-compiler-flags)
9. [Preprocessor Defines](#9-preprocessor-defines)
10. [Include Paths](#10-include-paths)
11. [Source Files](#11-source-files)
12. [Library Naming](#12-library-naming)
13. [Binding Code Generation](#13-binding-code-generation)
14. [Doc Data Generation](#14-doc-data-generation)
15. [Output Artifacts](#15-output-artifacts)
16. [Full Build Examples](#16-full-build-examples)

---

## 1. Overview

The project builds a **GDExtension shared library** that Godot loads at runtime. There are two compilation units:

1. **`libgodot-cpp` (static library)** — the godot-cpp wrapper library compiled from `godot-cpp/src/` and generated binding sources.
2. **User extension (shared library)** — the extension code in `src/` linked against `libgodot-cpp`.

The static library is then linked into the final shared library.

---

## 2. File Map and Roles

```
godot-cpp-template/
├── SConstruct                          ← Top-level build script (user extension)
├── custom.py                           ← (optional) persistent variable overrides
└── godot-cpp/
    ├── SConstruct                      ← godot-cpp sub-build entry
    ├── binding_generator.py            ← Generates C++ bindings from extension_api.json
    ├── build_profile.py                ← Trims extension_api.json to a subset
    ├── doc_source_generator.py         ← Converts doc XML → compressed .cpp
    ├── gdextension/
    │   ├── gdextension_interface.h     ← The GDExtension C API header
    │   └── extension_api.json          ← Full Godot API description
    ├── include/godot_cpp/              ← Public C++ headers
    ├── src/
    │   ├── godot.cpp
    │   ├── classes/                    ← wrapped.cpp, low_level.cpp, editor_plugin_registration.cpp
    │   ├── core/                       ← class_db, object, memory, method_bind, error_macros, print_string
    │   └── variant/                    ← All variant types (Vector2, Color, AABB, …)
    └── tools/
        ├── godotcpp.py                 ← Core build logic: options, generate(), GodotCPP builder
        ├── common_compiler_flags.py    ← C++ standard, optimization, debug, LTO, visibility flags
        ├── linux.py                    ← Linux platform configuration
        ├── macos.py                    ← macOS platform configuration
        ├── windows.py                  ← Windows (MSVC + MinGW) configuration
        ├── android.py                  ← Android NDK configuration
        ├── ios.py                      ← iOS configuration
        └── web.py                      ← WebAssembly (Emscripten) configuration
```

---

## 3. Build Flow

```
SConstruct (template)
  │
  ├─ reads custom.py for variable defaults
  │
  ├─ calls godot-cpp/SConstruct with the env
  │     │
  │     ├─ loads godotcpp tool (tools/godotcpp.py)
  │     ├─ registers all variables/options
  │     ├─ generates() the environment:
  │     │     ├─ detects platform + arch
  │     │     ├─ loads platform tool (e.g. linux.py)
  │     │     │     └─ calls common_compiler_flags.generate()
  │     │     │           ├─ sets C++17 standard
  │     │     │           ├─ sets optimization flags
  │     │     │           ├─ sets debug symbol flags
  │     │     │           ├─ sets LTO flags
  │     │     │           └─ sets visibility flags
  │     │     ├─ sets platform-specific CCFLAGS + LINKFLAGS + CPPDEFINES
  │     │     └─ assembles suffix string for output filenames
  │     │
  │     └─ GodotCPP() builder:
  │           ├─ (optionally) generates bindings via binding_generator.py
  │           └─ compiles godot-cpp sources → libgodot-cpp.<suffix>.a
  │
  ├─ appends src/ to CPPPATH
  ├─ collects src/*.cpp as user sources
  ├─ (optionally) generates doc_data.gen.cpp from doc_classes/*.xml
  ├─ builds SharedLibrary → bin/<platform>/<libname><suffix>.<ext>
  └─ installs library → project/bin/<platform>/
```

---

## 4. Build Options / Variables

These are configurable on the command line (e.g. `scons platform=linux target=editor`) or in `custom.py`.

### Core options

| Variable | Default | Choices | Description |
|---|---|---|---|
| `platform` | auto-detected | `linux` `macos` `windows` `android` `ios` `web` | Target platform |
| `target` | `template_debug` | `editor` `template_debug` `template_release` | Build variant |
| `arch` | auto-detected | `x86_64` `x86_32` `arm64` `arm32` `rv64` `ppc32` `ppc64` `wasm32` `universal` | CPU architecture |
| `precision` | `single` | `single` `double` | Floating-point precision |

### Optimization + Debug

| Variable | Default | Choices | Description |
|---|---|---|---|
| `optimize` | target-dependent | `none` `custom` `debug` `speed` `speed_trace` `size` | Optimization level |
| `lto` | `none` | `none` `auto` `thin` `full` | Link-Time Optimization |
| `debug_symbols` | `True` | `True` `False` | Include debug symbols |
| `dev_build` | `False` | `True` `False` | Enable `DEV_ENABLED` code paths |

### Feature flags

| Variable | Default | Description |
|---|---|---|
| `disable_exceptions` | `True` | Compile with `-fno-exceptions` |
| `symbols_visibility` | `hidden` | `auto`, `visible`, or `hidden` |
| `threads` | `True` | Enable `THREADS_ENABLED` |
| `use_hot_reload` | `True` (non-release) | Enable `HOT_RELOAD_ENABLED` |

### godot-cpp build

| Variable | Default | Description |
|---|---|---|
| `generate_bindings` | `False` | Force regeneration of C++ bindings |
| `generate_template_get_node` | `True` | Templated `get_node<T>()` helper |
| `build_library` | `True` | Whether to build `libgodot-cpp` |
| `build_profile` | `None` | JSON file to trim the Godot API (disables unused classes) |
| `gdextension_dir` | `godot-cpp/gdextension/` | Directory containing `gdextension_interface.h` + `extension_api.json` |
| `custom_api_file` | `None` | Override `extension_api.json` path |

### Toolchain (Linux / Windows)

| Variable | Default | Description |
|---|---|---|
| `use_llvm` | `False` | Use Clang instead of GCC (Linux/Windows) |
| `use_mingw` | `False` | Use MinGW instead of MSVC (Windows) |
| `use_static_cpp` | `True` | Statically link C++ runtime (MinGW) |
| `silence_msvc` | `True` | Silence MSVC stdout noise |
| `debug_crt` | `False` | Use MSVC debug CRT (`/MDd`) |
| `mingw_prefix` | `$MINGW_PREFIX` env var | MinGW toolchain prefix |

### Platform-specific

| Variable | Platform | Default | Description |
|---|---|---|---|
| `macos_deployment_target` | macOS | `default` | e.g. `10.15` |
| `macos_sdk_path` | macOS | `""` | Path to macOS SDK |
| `osxcross_sdk` | macOS | `darwin16` | OSXCross SDK version (cross-compile) |
| `ios_simulator` | iOS | `False` | Target iOS simulator |
| `ios_min_version` | iOS | `12.0` | Minimum iOS version |
| `IOS_TOOLCHAIN_PATH` | iOS | Xcode default | Xcode toolchain path |
| `android_api_level` | Android | `21` | Android API level (min 21) |
| `ANDROID_HOME` | Android | `$ANDROID_HOME` env | Android SDK root |

---

## 5. Platform Detection

Auto-detected from the host OS at build time:

```
sys.platform starts with "linux"    → platform = "linux"
sys.platform == "darwin"            → platform = "macos"
sys.platform in ("win32", "msys")   → platform = "windows"
```

Can always be overridden with `platform=<name>` on the command line (needed for cross-compilation).

---

## 6. Architecture Detection

When `arch` is left at its default (`""`), the build system selects an architecture automatically:

| Platform | Default arch |
|---|---|
| `macos` | `universal` (x86_64 + arm64 fat binary) |
| `ios` | `universal` |
| `android` | `arm64` |
| `web` | `wasm32` |
| all others | detected from host CPU |

**Host CPU detection** uses `platform.machine().lower()` matched against these aliases:

| Raw machine string | Resolved arch |
|---|---|
| `x64`, `amd64` | `x86_64` |
| `armv7`, `armv7l` | `arm32` |
| `armv8`, `arm64v8`, `aarch64` | `arm64` |
| `rv`, `riscv`, `riscv64` | `rv64` |
| `ppcle`, `ppc` | `ppc32` |
| `ppc64le` | `ppc64` |
| contains `"86"` | `x86_32` |

---

## 7. How `target` Affects the Build

Three boolean state variables are derived from `target`:

| State | Condition |
|---|---|
| `editor_build` | `target == "editor"` |
| `debug_features` | `target` is `editor` or `template_debug` |
| `use_hot_reload` | `target != "template_release"` (unless manually overridden) |

### Default optimization per target

| `target` | `dev_build` | Default `optimize` |
|---|---|---|
| `template_release` | `no` | `speed` → `-O3` |
| `editor` / `template_debug` | `no` | `speed_trace` → `-O2` |
| any | `yes` | `none` → `-O0` |

---

## 8. Compiler Flags

### C++ standard

- GCC/Clang: `-std=c++17`
- MSVC: `/std:c++17`

### Exception handling

When `disable_exceptions=yes` (default):
- GCC/Clang: `-fno-exceptions`
- MSVC: defines `_HAS_EXCEPTIONS=0`

When `disable_exceptions=no` on MSVC: `/EHsc`

### Symbol visibility (non-MSVC only)

| `symbols_visibility` | Flags added |
|---|---|
| `hidden` (default) | `-fvisibility=hidden` (CCFLAGS + LINKFLAGS) |
| `visible` | `-fvisibility=default` (CCFLAGS + LINKFLAGS) |
| `auto` | none |

### Debug symbols

| Condition | GCC/Clang | MSVC |
|---|---|---|
| `debug_symbols=yes`, `dev_build=yes` | `-gdwarf-4 -g3` | `/Zi /FS` + link `/DEBUG:FULL` |
| `debug_symbols=yes`, `dev_build=no` | `-gdwarf-4 -g2` | `/Zi /FS` + link `/DEBUG:FULL` |
| `debug_symbols=no`, non-Apple | `-s` (strip) | — |
| `debug_symbols=no`, Apple | `-Wl,-S -Wl,-x -Wl,-dead_strip` | — |

### Optimization flags

| `optimize=` | GCC/Clang | MSVC |
|---|---|---|
| `speed` | `-O3` | `/O2` + `/OPT:REF` |
| `speed_trace` | `-O2` | `/O2` + `/OPT:REF /OPT:NOICF` |
| `size` | `-Os` | `/O1` + `/OPT:REF` |
| `debug` | `-Og` | `/Od` |
| `none` | `-O0` | `/Od` |
| `custom` | (nothing) | (nothing) |

### LTO flags

| `lto=` | GCC/Clang | MSVC |
|---|---|---|
| `thin` | `-flto=thin` (CCFLAGS + LINKFLAGS) | `-flto=thin` (LLVM only) |
| `full` | `-flto` (CCFLAGS + LINKFLAGS) | `/GL` (CCFLAGS) + `/LTCG` (ARFLAGS + LINKFLAGS) |

LTO `auto` defaults per platform: Linux → `full`, macOS → `none`, Windows MSVC → `none`, Windows MinGW → `full`, Android → `none`, iOS → `none`, Web → `full`.

### Platform-specific flags

#### Linux

```
CCFLAGS:   -fPIC  -Wwrite-strings
           + arch flags (see below)
           + -fno-gnu-unique  (if use_hot_reload=yes AND NOT llvm)
LINKFLAGS: -Wl,-R,'$$ORIGIN'
           + arch flags

Per-arch flags:
  x86_64:  -m64 -march=x86-64
  x86_32:  -m32 -march=i686
  arm64:   -march=armv8-a
  rv64:    -march=rv64gc
```

#### Windows (MSVC)

```
CCFLAGS:   /utf-8
           + /MT  (static CRT, default)
           | /MD  (dynamic CRT, use_static_cpp=no)
           | /MDd (debug CRT, debug_crt=yes)
LINKFLAGS: /WX
CPPDEFINES: TYPED_METHOD_BIND  NOMINMAX
```

#### Windows (MinGW)

```
CCFLAGS:   -Wwrite-strings
LINKFLAGS: -Wl,--no-undefined
           + -static -static-libgcc -static-libstdc++  (use_static_cpp=yes)
SHLIBPREFIX: ""
SHLIBSUFFIX: ".dll"
```

#### macOS

```
Compiler: clang / clang++
CCFLAGS:  -arch x86_64 -arch arm64  (universal)
          -arch <arch>               (single-arch)
          + -mmacosx-version-min=<ver>  (if set)
          + -isysroot <sdk>             (if set)
LINKFLAGS: same arch/min-version/isysroot flags
           + -framework Cocoa
           + -Wl,-undefined,dynamic_lookup
CPPDEFINES: MACOS_ENABLED  UNIX_ENABLED
SHLIBSUFFIX: ".dylib"
```

#### Android (NDK clang, API ≥ 21)

```
Compiler: NDK clang/clang++
  (from $ANDROID_HOME/ndk/<version>/toolchains/llvm/prebuilt/<host>/bin/)
CCFLAGS:  --target=<triple><api>  -march=<march>  -fPIC
          arm32 extra: -mfpu=neon
          x86_32 extra: -mstackrealign
LINKFLAGS: --target=...  -march=...
CPPDEFINES: ANDROID_ENABLED  UNIX_ENABLED
SHLIBSUFFIX: ".so"

Arch → triple mapping:
  arm32   → armv7a-linux-androideabi
  arm64   → aarch64-linux-android
  x86_32  → i686-linux-android
  x86_64  → x86_64-linux-android
```

#### iOS

```
Compiler: Xcode clang
CCFLAGS:  -arch arm64 (or x86_64/universal)
          -isysroot <sdk>
          -miphoneos-version-min=12.0
          (or -mios-simulator-version-min=... for simulator)
LINKFLAGS: -arch ...  -isysroot <sdk>  -F<sdk>
CPPDEFINES: IOS_ENABLED  UNIX_ENABLED
SHLIBSUFFIX: ".dylib"
```

#### Web (Emscripten)

```
Compiler: emcc / em++
CCFLAGS:  -sSIDE_MODULE=1
          -sSUPPORT_LONGJMP='wasm'
          + -sUSE_PTHREADS=1  (if threads=yes)
LINKFLAGS: -sSIDE_MODULE=1  -sWASM_BIGINT  -sSUPPORT_LONGJMP='wasm'
           + -sUSE_PTHREADS=1  (if threads=yes)
CPPDEFINES: WEB_ENABLED  UNIX_ENABLED
SHLIBSUFFIX: ".wasm"
```

---

## 9. Preprocessor Defines

| Define | Set when |
|---|---|
| `GDEXTENSION` | **Always** — universal GDExtension marker |
| `THREADS_ENABLED` | `threads=yes` (default) |
| `HOT_RELOAD_ENABLED` | `use_hot_reload=yes` (non-release targets by default) |
| `TOOLS_ENABLED` | `target=editor` |
| `DEBUG_ENABLED` | `target=editor` or `target=template_debug` |
| `DEBUG_METHODS_ENABLED` | same as above |
| `DEV_ENABLED` | `dev_build=yes` |
| `NDEBUG` | `dev_build=no` (default) |
| `REAL_T_IS_DOUBLE` | `precision=double` |
| `LINUX_ENABLED` | Linux platform |
| `UNIX_ENABLED` | Linux, macOS, Android, iOS, Web |
| `MACOS_ENABLED` | macOS platform |
| `WINDOWS_ENABLED` | Windows platform |
| `ANDROID_ENABLED` | Android platform |
| `IOS_ENABLED` | iOS platform |
| `WEB_ENABLED` | Web platform |
| `TYPED_METHOD_BIND` | Windows MSVC only |
| `NOMINMAX` | Windows MSVC only |
| `_HAS_EXCEPTIONS=0` | `disable_exceptions=yes` on MSVC |

---

## 10. Include Paths

In order of search priority:

| Path | Contents |
|---|---|
| `godot-cpp/gdextension/` | `gdextension_interface.h` — the raw C API header |
| `godot-cpp/include/` | Public C++ wrappers (`godot_cpp/*.hpp`) |
| `godot-cpp/gen/include/` | Generated class binding headers |
| `src/` | User extension headers |

---

## 11. Source Files

### godot-cpp static library

All source files under `godot-cpp/src/`:

```
godot-cpp/src/godot.cpp
godot-cpp/src/classes/wrapped.cpp
godot-cpp/src/classes/low_level.cpp
godot-cpp/src/classes/editor_plugin_registration.cpp
godot-cpp/src/core/class_db.cpp
godot-cpp/src/core/error_macros.cpp
godot-cpp/src/core/memory.cpp
godot-cpp/src/core/method_bind.cpp
godot-cpp/src/core/object.cpp
godot-cpp/src/core/print_string.cpp
godot-cpp/src/variant/aabb.cpp
godot-cpp/src/variant/basis.cpp
godot-cpp/src/variant/callable_bind.cpp
godot-cpp/src/variant/callable_custom.cpp
godot-cpp/src/variant/char_string.cpp
godot-cpp/src/variant/color.cpp
godot-cpp/src/variant/packed_arrays.cpp
godot-cpp/src/variant/plane.cpp
godot-cpp/src/variant/projection.cpp
godot-cpp/src/variant/quaternion.cpp
godot-cpp/src/variant/rect2.cpp
godot-cpp/src/variant/rect2i.cpp
godot-cpp/src/variant/transform2d.cpp
godot-cpp/src/variant/transform3d.cpp
godot-cpp/src/variant/variant_internal.cpp
godot-cpp/src/variant/variant.cpp
godot-cpp/src/variant/vector2.cpp  vector2i.cpp
godot-cpp/src/variant/vector3.cpp  vector3i.cpp
godot-cpp/src/variant/vector4.cpp  vector4i.cpp

+ godot-cpp/gen/src/classes/*.cpp  ← all generated Godot class bindings
```

### User extension shared library

```
src/*.cpp                       ← all user source files
src/gen/doc_data.gen.cpp        ← (only for editor + template_debug targets, if doc_classes/*.xml exist)
```

The final shared library is linked against `libgodot-cpp.<suffix>.a`.

---

## 12. Library Naming

### Suffix format

The suffix string is assembled as:

```
.<platform>.<target>[.dev][.double].<arch>[.simulator][.nothreads]
```

- `.dev` is appended when `dev_build=yes`
- `.double` is appended when `precision=double`
- `.simulator` is appended for iOS simulator builds
- `.nothreads` is appended when `threads=no`

Examples:
```
.linux.template_debug.x86_64
.linux.editor.x86_64
.windows.template_release.x86_64
.macos.template_debug.universal
.android.template_debug.arm64
.web.template_release.wasm32.nothreads
.linux.template_debug.dev.x86_64
```

### godot-cpp static library

```
lib/libgodot-cpp<suffix>.a         (Linux, macOS, Android, Web)
lib/libgodot-cpp<suffix>.lib       (Windows MSVC)
```

### User extension shared library

The template `SConstruct` strips `.dev` and `.universal` from the suffix before naming the output file:

```
bin/<platform>/<SHLIBPREFIX><libname><suffix><SHLIBSUFFIX>
```

Installed copy:
```
project/bin/<platform>/<filename>
```

**`SHLIBPREFIX` and `SHLIBSUFFIX` per platform:**

| Platform | Prefix | Suffix |
|---|---|---|
| Linux | `lib` | `.so` |
| macOS | `lib` | `.dylib` |
| Windows MSVC | `` | `.dll` |
| Windows MinGW | `` | `.dll` |
| Android | `lib` | `.so` |
| iOS | `lib` | `.dylib` |
| Web | `lib` | `.wasm` |

**Full filename examples** (with `libname = "EXTENSION-NAME"`):

```
bin/linux/libEXTENSION-NAME.linux.template_debug.x86_64.so
bin/windows/EXTENSION-NAME.windows.editor.x86_64.dll
bin/macos/libEXTENSION-NAME.macos.template_release.dylib
bin/android/libEXTENSION-NAME.android.template_debug.arm64.so
bin/web/libEXTENSION-NAME.web.template_debug.wasm32.wasm
```

---

## 13. Binding Code Generation

The `GodotCPPBindings` builder calls `binding_generator.py` to produce C++ source and header files under `godot-cpp/gen/`.

**Inputs:**
- `godot-cpp/gdextension/extension_api.json` — full Godot API description
- `godot-cpp/gdextension/gdextension_interface.h`
- `binding_generator.py`

**Outputs (under `godot-cpp/gen/`):**
- `gen/include/godot_cpp/classes/*.hpp` — one header per Godot class
- `gen/src/classes/*.cpp` — one source per Godot class

**Parameters that affect generation:**
- `precision` (`single`/`double`) — selects 32-bit or 64-bit float paths
- `generate_template_get_node` — adds `get_node<T>()` template helper
- `build_profile` — calls `generate_trimmed_api()` to drop unused classes, reducing compile time

Bindings are regenerated when `extension_api.json` or `gdextension_interface.h` changes, or when `generate_bindings=yes` is forced.

---

## 14. Doc Data Generation

Only applies when `target` is `editor` or `template_debug` **and** `doc_classes/*.xml` files exist.

`doc_source_generator.py` reads all XML files in `doc_classes/`, concatenates and zlib-compresses them (`Z_BEST_COMPRESSION`), then emits `src/gen/doc_data.gen.cpp` containing:

```cpp
static const unsigned char _doc_data_compressed[] = { /* compressed bytes */ };
static godot::internal::DocDataRegistration _doc_data_registration(
    _doc_data_compressed, sizeof(_doc_data_compressed), /* uncompressed size */
);
```

The `DocDataRegistration` object auto-registers the documentation with Godot's internal class reference system at runtime.

This source file is compiled into the extension shared library for `editor`/`template_debug` targets only. Release builds omit it entirely.

If the `GodotCPPDocData` builder is absent (pre-4.3 godot-cpp), the error is silently caught and doc data is skipped.

---

## 15. Output Artifacts

| Artifact | Path |
|---|---|
| godot-cpp static library | `godot-cpp/bin/libgodot-cpp<suffix>.a` |
| Extension shared library | `bin/<platform>/<libname><suffix>.<ext>` |
| Installed copy | `project/bin/<platform>/<libname><suffix>.<ext>` |
| Generated bindings headers | `godot-cpp/gen/include/` |
| Generated bindings sources | `godot-cpp/gen/src/` |
| Doc data source | `src/gen/doc_data.gen.cpp` (when applicable) |
| Compilation database | `compile_commands.json` (when `compiledb=yes`) |

---

## 16. Full Build Examples

### Linux, debug (default)

```bash
scons platform=linux target=template_debug
```

Effective configuration:
```
Defines:   GDEXTENSION DEBUG_ENABLED DEBUG_METHODS_ENABLED HOT_RELOAD_ENABLED
           THREADS_ENABLED NDEBUG LINUX_ENABLED UNIX_ENABLED
CXXFLAGS:  -std=c++17 -fno-exceptions -fvisibility=hidden
           -fPIC -Wwrite-strings -fno-gnu-unique
           -m64 -march=x86-64
           -gdwarf-4 -g2 -O2
LINKFLAGS: -Wl,-R,'$$ORIGIN' -m64 -march=x86-64 -fvisibility=hidden
Static:    godot-cpp/bin/libgodot-cpp.linux.template_debug.x86_64.a
Shared:    bin/linux/libEXTENSION-NAME.linux.template_debug.x86_64.so
Installed: project/bin/linux/libEXTENSION-NAME.linux.template_debug.x86_64.so
```

### Linux, release

```bash
scons platform=linux target=template_release
```

Effective configuration:
```
Defines:   GDEXTENSION THREADS_ENABLED NDEBUG LINUX_ENABLED UNIX_ENABLED
CXXFLAGS:  -std=c++17 -fno-exceptions -fvisibility=hidden
           -fPIC -Wwrite-strings
           -m64 -march=x86-64
           -gdwarf-4 -g2 -O3 -s
Shared:    bin/linux/libEXTENSION-NAME.linux.template_release.x86_64.so
```

### Windows (MSVC), editor

```bash
scons platform=windows target=editor
```

Effective configuration:
```
Defines:   GDEXTENSION DEBUG_ENABLED DEBUG_METHODS_ENABLED HOT_RELOAD_ENABLED
           TOOLS_ENABLED THREADS_ENABLED NDEBUG WINDOWS_ENABLED
           TYPED_METHOD_BIND NOMINMAX
CXXFLAGS:  /std:c++17 /utf-8 /MT /Zi /FS /O2
LINKFLAGS: /WX /DEBUG:FULL
Shared:    bin/windows/EXTENSION-NAME.windows.editor.x86_64.dll
```

### Android, arm64, debug

```bash
scons platform=android target=template_debug arch=arm64
```

Effective configuration:
```
Defines:   GDEXTENSION DEBUG_ENABLED DEBUG_METHODS_ENABLED HOT_RELOAD_ENABLED
           THREADS_ENABLED NDEBUG ANDROID_ENABLED UNIX_ENABLED
CXXFLAGS:  -std=c++17 -fno-exceptions -fvisibility=hidden
           --target=aarch64-linux-android21 -march=armv8-a -fPIC
           -gdwarf-4 -g2 -O2
LINKFLAGS: --target=aarch64-linux-android21 -march=armv8-a -fvisibility=hidden
Shared:    bin/android/libEXTENSION-NAME.android.template_debug.arm64.so
```

### Web (Emscripten), release, no threads

```bash
scons platform=web target=template_release threads=no
```

Effective configuration:
```
Defines:   GDEXTENSION NDEBUG WEB_ENABLED UNIX_ENABLED
CXXFLAGS:  -std=c++17 -fno-exceptions -fvisibility=hidden
           -sSIDE_MODULE=1 -sSUPPORT_LONGJMP='wasm'
           -O3
LINKFLAGS: -sSIDE_MODULE=1 -sWASM_BIGINT -sSUPPORT_LONGJMP='wasm'
Shared:    bin/web/libEXTENSION-NAME.web.template_release.wasm32.nothreads.wasm
```
