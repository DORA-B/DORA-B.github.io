---
title: Compile-llvm-test-suit-story-2
date: 2025-10-05
categories: [Environment]
tags: [linux]     # TAG names should always be lowercase
published: false
---

> This story continues from the compilation of llvm-test-suite, I want to use different compilation mechanisms from the original ones, and aslo, more DIY styles of compilation, including different compilation flags, more steps, and additional steps (generate .s for each app), and this blog is likely a more detailed description of cmake files in llvm-test-suite.

# CMake Summary: LLVM Test Suite Cross-Compilation Pipeline

## Overview

This document summarizes the technical insights gained from implementing a custom ARM cross-compilation pipeline for the LLVM Test Suite, focusing on CMake architecture, cache mechanisms, and hierarchical build management.

## CMake Architecture in LLVM Test Suite

### Hierarchical CMakeLists.txt Structure

The LLVM Test Suite uses a sophisticated hierarchical CMake structure:

```
llvm-test-suite/
├── CMakeLists.txt                    # Root configuration
├── cmake/
│   ├── modules/
│   │   └── TestSuite.cmake          # Core test definitions
│   └── caches/
│       └── ARM-EABI-LLM.cmake       # Custom cache file
├── SingleSource/
│   ├── CMakeLists.txt               # SingleSource configuration
│   ├── Benchmarks/
│   │   ├── CMakeLists.txt           # Benchmark category
│   │   ├── CoyoteBench/
│   │   │   └── CMakeLists.txt       # Specific benchmark group
│   │   ├── Dhrystone/
│   │   │   └── CMakeLists.txt
│   │   └── [other benchmark dirs]
│   └── [other categories]
└── MultiSource/
    └── [similar structure]
```

### CMake Cache Mechanism

#### Custom Cache File: `cmake/caches/ARM-EABI-LLM.cmake`

```cmake
# Cross-compilation configuration
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR arm)

# Toolchain configuration
set(CMAKE_C_COMPILER clang-17)
set(CMAKE_CXX_COMPILER clang++-17)

# Target architecture flags
set(CMAKE_C_FLAGS "--target=arm-linux-gnueabi --sysroot=/usr/arm-linux-gnueabi" CACHE STRING "")
set(CMAKE_CXX_FLAGS "--target=arm-linux-gnueabi --sysroot=/usr/arm-linux-gnueabi" CACHE STRING "")

# Custom compilation pipeline flags
set(COMMON_FLAGS "-DNDEBUG -march=armv5te -mfloat-abi=soft -marm -O2 -g -gdwarf-4")
set(DISABLE_OPTIMIZATION_FLAGS "-fno-vectorize -fno-slp-vectorize -fno-unroll-loops")
set(ARM_SPECIFIC_FLAGS "-mllvm -disable-early-ifcvt -mllvm -ifcvt-limit=0")
```

#### Cache Loading Process

```bash
# Cache is loaded during configuration
cmake -S . -B build-3stage -C cmake/caches/ARM-EABI-LLM.cmake -G Ninja

# Cache variables override default CMake behavior
# CMAKE_C_FLAGS, CMAKE_CXX_FLAGS are set before any CMakeLists.txt processing
ninja -C build-3stage -v
```

### TestSuite.cmake Module Functions

#### Core Test Definition Functions

```cmake
# From cmake/modules/TestSuite.cmake
llvm_test_executable(target_name source_files...)
llvm_test_run(target_name)
llvm_multisource(target_name source_files...)
```

#### Custom Pipeline Integration

The custom pipeline integrates with TestSuite.cmake through:

1. **Object file generation**: Standard CMake compilation produces `.o` files
2. **Custom linking stage**: ARM-specific partial linking with `arm-linux-gnueabi-ld`
3. **Final binary creation**: ARM GCC linking with cross-compilation libraries

### Build Directory Structure

#### Standard LLVM Test Suite Structure
```
build-3stage/
├── CMakeCache.txt                   # Generated cache
├── CMakeFiles/                      # CMake metadata
├── SingleSource/
│   └── Benchmarks/
│       └── CoyoteBench/
│           ├── CMakeFiles/
│           │   └── huffbench.dir/
│           │       ├── huffbench.c.o          # Standard object
│           │       └── huffbench.c.o.backup   # Our backup
│           ├── huffbench                       # Standard binary
│           ├── huffbench.combined.o            # Our combined object
│           ├── huffbench.combined.s            # Our disassembly
│           └── rewriter_files/                 # Our intermediate files
│               ├── huffbench_original.s
│               ├── huffbench_revised.s
│               └── huffbench_revised.o
```

## Custom Compilation Pipeline Integration

### 3-Stage Compilation Process

#### Stage 1: Source to Assembly
```bash
# Using clang with exact flags from CMake configuration
/usr/bin/clang-17 $COMMON_FLAGS $extra_flags -S -o "$asm_file" "$source_file"
```

#### Stage 2: Assembly Transformation
```bash
# Custom predicated instruction rewriting
python3 bin_rewriter.py "$original_asm" "$revised_asm"
```

#### Stage 3: Object File Replacement and Linking
```bash
# Compile revised assembly
/usr/bin/clang-17 $COMMON_FLAGS -c -o "$revised_obj" "$revised_asm"

# Replace original CMake-generated object
cp "$revised_obj" "$original_cmake_obj"

# Custom partial linking (bypassing CMake)
/usr/bin/arm-linux-gnueabi-ld -r -o "$combined_obj" "$original_cmake_obj"

# Custom final linking (bypassing CMake)
arm-linux-gnueabi-gcc $arm_link_flags "$combined_obj" -o "$final_binary" $arm_libs

# Generate disassembly for analysis
/usr/bin/arm-linux-gnueabi-objdump --source --line-numbers \
    --disassemble --reloc --demangle "$combined_obj" > "$combined_asm"
```

### CMake Integration Points

#### 1. Object File Discovery
```bash
# CMake generates objects in predictable locations
build_dir="build-3stage/$source_dir"
cmake_subdir="CMakeFiles/$app_name.dir"

# Multiple naming conventions supported
possible_objects=(
    "$build_dir/$cmake_subdir/${name_no_ext}.o"
    "$build_dir/$cmake_subdir/${name_no_ext}.c.o"
    "$build_dir/$cmake_subdir/${name_no_ext}.cpp.o"
)
```

#### 2. Compiler Flag Inheritance
```bash
# Flags are extracted from buildlog.log (CMake's actual commands)
# Common flags from cache configuration
COMMON_FLAGS="--target=arm-linux-gnueabi --sysroot=/usr/arm-linux-gnueabi..."

# Application-specific flags
if [[ "$compiler" == "clang++-17" ]]; then
    extra_flags="-Wno-deprecated"
    if [[ "$app_name" == "spirit" ]]; then
        extra_flags="$extra_flags -std=gnu++14 -pthread"
    fi
elif [[ "$app_name" == "dry" ]]; then
    extra_flags="-Wno-implicit-int"
fi
```

## Key Technical Insights

### 1. CMake Cache Override Mechanism

The cache file (`-C cmake/caches/ARM-EABI-LLM.cmake`) sets variables **before** any CMakeLists.txt processing:

- `CMAKE_C_COMPILER` and `CMAKE_CXX_COMPILER` are locked in
- `CMAKE_C_FLAGS` and `CMAKE_CXX_FLAGS` become baseline for all compilations
- Cross-compilation settings (`CMAKE_SYSTEM_NAME`, `CMAKE_SYSTEM_PROCESSOR`) affect all target generation

### 2. Hierarchical Flag Inheritance

Flags flow down the hierarchy:
```
Root CMakeLists.txt (applies cache)
    ↓
SingleSource/CMakeLists.txt (adds category flags)
    ↓
Benchmarks/CMakeLists.txt (adds benchmark flags)
    ↓
CoyoteBench/CMakeLists.txt (adds specific flags)
    ↓
Individual target compilation
```

### 3. Build System Bypass Strategy

Our pipeline **bypasses** CMake's linking stages while **leveraging** its compilation stages:

- **Keep**: CMake's source → object compilation (with custom flags)
- **Replace**: Object file content (with rewritten assembly)
- **Bypass**: CMake's linking (use custom ARM toolchain)
- **Preserve**: CMake's build directory structure

### 4. Toolchain Selection Logic

```bash
# CMake configuration determines base compiler
CMAKE_C_COMPILER=clang-17
CMAKE_CXX_COMPILER=clang++-17

# But final linking uses different tools
# Partial linking: arm-linux-gnueabi-ld
# Final linking: arm-linux-gnueabi-gcc
# Disassembly: arm-linux-gnueabi-objdump
```

## Build Process Integration

### Standard LLVM Test Suite Build
```bash
mkdir build && cd build
cmake .. -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++
make
make check
```

### Our Custom Pipeline Build
```bash
# 1. Configure with custom cache
cmake -S . -B build-3stage -C cmake/caches/ARM-EABI-LLM.cmake -G Ninja

# 2. Build targets to generate objects
ninja huffbench chomp dry  # etc.

# 3. Run custom rewriting pipeline
./rewrite_predicated_apps.sh

# 4. Analyze results
./batch_analyze_mappings.sh
```

## Debug Information and Analysis

### DWARF-4 Debug Info Preservation
```bash
# Flags ensure debug info survives through pipeline
-g -gdwarf-4 -fno-omit-frame-pointer
```

### Assembly Analysis Integration
```bash
# Source mapping preserved through:
objdump --source --line-numbers --disassemble --reloc --demangle
```

## Lessons Learned

### 1. CMake Cache Power
- Cache files can completely override default behavior
- Variables set in cache are **immutable** during configuration
- Cache is the correct way to set cross-compilation defaults

### 2. Build System Flexibility
- CMake's object generation can be preserved while customizing linking
- Directory structure conventions enable predictable file locations
- Multiple toolchains can coexist in the same build

### 3. Flag Management Complexity
- Application-specific flags require careful discovery from buildlog.log
- Compiler detection logic must handle both C and C++ variants
- Error messages guide flag requirements (e.g., `-Wno-implicit-int`)

### 4. Pipeline Integration Strategy
- **Minimal invasiveness**: Preserve CMake's strengths, override only what's necessary
- **Debugging support**: Preserve all intermediate files for analysis
- **Verification**: Multiple analysis points ensure transformation correctness


# CMake rule placeholders can be used in cache/toolchain overrides

These are expanded by CMake when overriding rule-templates like `CMAKE_C_COMPILE_OBJECT`, `CMAKE_CXX_COMPILE_OBJECT`, `CMAKE_<LANG>_LINK_EXECUTABLE`, etc.

Common, portable placeholders:

* `<CMAKE_C_COMPILER>`, `<CMAKE_CXX_COMPILER>` — the actual compiler exe
    
* `<DEFINES>` — `-D…` defines for this TU
    
* `<INCLUDES>` — `-I…` include paths
    
* `<FLAGS>` — language + config + target flags (e.g., `-O2 -g …`)
    
* `<SOURCE>` — the source file path
    
* `<OBJECT>` — the output object file path
    
* `<OBJECT_DIR>` — directory for the object
    
* `<TARGET>` — the output (binary/library) path for link rules
    
* `<LINK_FLAGS>` — link flags set on the target
    
* `<LINK_LIBRARIES>` — resolved libs and paths
    
* `<CMAKE_C_LINK_FLAGS>` / `<CMAKE_CXX_LINK_FLAGS>` — global link flags for the language
    

Generator-specific / super useful:

* `<DEPFILE>` — depfile path (Ninja/Make). Pair with `-MMD -MF <DEPFILE> -MT <OBJECT>` in compile rules.
    
* `<TARGET_PDB>`, `<TARGET_SONAME_FILE>`, … — mainly MSVC/ELF specifics.
    

Tip (generator-agnostic pipelines): avoid `cmd1 && cmd2 && cmd3` directly in a rule (Make vs Ninja shell handling). Use a tiny wrapper:

```cmake
set(CMAKE_C_COMPILE_OBJECT
  "${CMAKE_COMMAND} -E sh -c \"<CMAKE_C_COMPILER> <DEFINES> <INCLUDES> <FLAGS> -MMD -MF <DEPFILE> -MT <OBJECT> -S -o <OBJECT>.orig.s <SOURCE> && arm_predicate_rewriter <OBJECT>.orig.s > <OBJECT>.s && <CMAKE_C_COMPILER> <INCLUDES> <FLAGS> -c <OBJECT>.s -o <OBJECT>\"")
```

`cmake -E sh -c` forces execution via a shell across generators.

How to list targets (CMake vs Ninja)
====================================

CMake (generator-agnostic)
--------------------------

* **List all build targets:**
    
    ```bash
    cmake --build build-armv5te --target help
    ```
    
    Works for Ninja/Make/VS; prints a friendly list of CMake targets (executables, libs, utility/phony).
    
* **Show verbose commands for a given target:**
    
    ```bash
    cmake --build build-armv5te --target huffbench -- -v
    ```
    
    `--` passes native build tool options (`-v` means “verbose” for Ninja/Make).
    
* **Parallelism (portable):**
    
    ```bash
    cmake --build build-armv5te --parallel 16
    ```
    

Ninja (extra introspection superpowers)
---------------------------------------

* **List all targets (with categories):**
    
    ```bash
    ninja -C build-armv5te -t targets all
    ```
    
* **See the exact command lines Ninja will run:**
    
    ```bash
    ninja -C build-armv5te -t commands huffbench
    ```
    
* **Dependency graph (DOT):**
    
    ```bash
    ninja -C build-armv5te -t graph huffbench > graph.dot
    dot -Tpng graph.dot -o graph.png
    ```
    
* **Header deps Ninja knows about (from `.d` files):**
    
    ```bash
    ninja -C build-armv5te -t deps SingleSource/.../huffbench.c.o
    ```
    
* **Why Ninja rebuilt something:**
    
    ```bash
    ninja -C build-armv5te -d explain huffbench
    ```
    
* **Compile database (for tooling):**
    
    ```bash
    ninja -C build-armv5te -t compdb cxx cc > compile_commands.json
    ```
    
    (Or set `-DCMAKE_EXPORT_COMPILE_COMMANDS=ON` at configure time.)
    
* **Verbose build:**
    
    ```bash
    ninja -C build-armv5te -v huffbench
    ```
    

Practical patterns for 3-stage (.c→.s→rewrite→.o) pipeline
===============================================================

Keep header-dependency tracking intact
--------------------------------------

Make the **first stage** emit the depfile CMake/Ninja expects:

```cmake
set(CMAKE_C_COMPILE_OBJECT
  "<CMAKE_C_COMPILER> <DEFINES> <INCLUDES> <FLAGS> -MMD -MF <DEPFILE> -MT <OBJECT> -S -o <OBJECT>.orig.s <SOURCE> && "
  "arm_predicate_rewriter <OBJECT>.orig.s > <OBJECT>.s && "
  "<CMAKE_C_COMPILER> <INCLUDES> <FLAGS> -c <OBJECT>.s -o <OBJECT>")
```

Now header edits trigger a correct rebuild of the object.

Scope target-only flags to compile, not assemble
------------------------------------------------

Put `-mllvm …` switches **only** on the compilation stage (the assembler ignores them). If split rules:

* Stage 1: add `-mllvm …`
    
* Stage 3: **omit** them
    

Ninja vs. CMake help: what’s different?
=======================================

* `cmake --build … --target help` lists CMake-level targets (logical graph).
    
* `ninja -t targets all` lists **Ninja rules** including phony dirs/utility rules CMake generated—more granular and sometimes more noisy.
    
* For day-to-day work: use `cmake --build` for portability; when debugging or exploring the DAG, switch to `ninja -t …`.
    

one-liners
====================================

```bash
# List targets (portable)
cmake --build build-3stage --target help

# Build a single benchmark verbosely
cmake --build build-3stage --target huffbench -- -v

# Same via Ninja + why it rebuilt
ninja -C build-3stage -d explain -v huffbench

# Show the exact command used to build huffbench
ninja -C build-3stage -t commands huffbench

# Visualize the dependency graph
ninja -C build-3stage -t graph huffbench | dot -Tsvg > huffbench_dag.svg

# Export compile_commands.json for tooling
cmake -S . -B build-3stage -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
ln -sf build-3stage/compile_commands.json .
```

Pitfalls
===================================

* Multi-command rule lines behave differently across generators; prefer `cmake -E sh -c` or a wrapper script.
    
* Global overrides (e.g., `CMAKE_<LANG>_LINK_EXECUTABLE`) affect **everything** (including host tools). Scope changes or use two builds when needed.
    
* Keep depfiles (`<DEPFILE>` with `-MMD -MF`) or header edits won’t retrigger builds.
    
* Use `--` to pass native flags (`-v`, `-k`, etc.) through `cmake --build`.

