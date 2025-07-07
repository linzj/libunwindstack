# Java vs Native Stack Unwinding in libunwindstack

## Overview

This document analyzes how libunwindstack handles Java stack traces differently from native stack traces. Based on the Android libunwindstack codebase, here are the key differences and mechanisms.

## Core Differences

### 1. **Program Counter Handling**

#### Native Stack Unwinding:
- Uses standard CPU program counter (PC) from processor registers
- PC points directly to machine instructions in executable code sections
- Unwinding follows standard calling conventions and frame pointers/DWARF info

#### Java Stack Unwinding:
- Uses a **"dex_pc"** (DEX program counter) - a virtual program counter
- `dex_pc` is a magic register defined in the Mterp (Method interpreter) 
- Points to bytecode instructions in DEX files, not native machine code
- Stored in `regs_->dex_pc()` and managed separately from native PC

### 2. **Frame Injection Mechanism**

The most significant difference is how Java frames are handled in the unwinding process:

#### Virtual Frame Injection:
```cpp
// From Unwinder.cpp:193-203
if (regs_->dex_pc() != 0) {
    // Add a frame to represent the dex file.
    FillInDexFrame();
    // Clear the dex pc so that we don't repeat this frame later.
    regs_->set_dex_pc(0);
}
```

**Java frames are injected as virtual frames** that represent the DEX bytecode execution context. This creates a stack trace like:
```
#7 pc 0015fa20 core.vdex   java.util.Arrays.binarySearch+8
#8 pc 006b1ba1 libartd.so  ExecuteMterpImpl+14625
#9 pc 0039a1ef libartd.so  art::interpreter::Execute+719
```

### 3. **Symbol Resolution**

#### Native Symbol Resolution:
- Uses ELF symbol tables, DWARF debug info, and GNU debug data
- Resolves function names from compiled native libraries
- Handles C++ symbol demangling

#### Java Symbol Resolution:
```cpp
// From Unwinder.cpp:83
dex_files_->GetFunctionName(maps_, dex_pc, &frame->function_name, &frame->function_offset);
```

Java symbol resolution:
- Uses DEX file format and ART API (`art_api::dex::DexFile`)
- Resolves Java method names from DEX metadata
- Method lookup based on bytecode offset within DEX file
- Returns qualified Java method names (e.g., `java.util.Arrays.binarySearch`)

### 4. **Memory Mapping and File Handling**

#### Native Code:
- Maps standard ELF executables and shared libraries
- Uses virtual memory addresses directly
- Handles standard executable sections (.text, .data, etc.)

#### Java Code:
- Maps DEX files (`.dex`, `.vdex`, `.oat` files)  
- Converts absolute `dex_pc` to file-relative offset:
  ```cpp
  uint64_t dex_offset = dex_pc - base_addr_;
  ```
- Handles DEX-specific file structures and metadata

### 5. **Caching and Performance**

#### Java-Specific Optimizations:
- **File-level caching**: DEX files are cached to avoid reloading expensive lookup tables
- **Weak pointer cache**: Uses `std::weak_ptr` to cache DEX file APIs without keeping files alive
- **Thread-safe symbol cache**: Caches method lookups by end offset for performance

```cpp
// DEX file caching mechanism
using MappedFileKey = std::tuple<std::string, uint64_t, uint64_t>;  // (path, offset, size)
static std::map<MappedFileKey, std::weak_ptr<DexFileApi>> g_mapped_dex_files;
```

### 6. **Error Handling and Warnings**

Java unwinding introduces specific error conditions:
- `WARNING_DEX_PC_NOT_IN_MAP`: DEX PC doesn't correspond to any mapped region
- Special handling for deleted DEX files (when apps are background-optimized)
- Fallback to memory copy when disk-based DEX access fails

### 7. **Integration with Native Stack**

Java frames are **interleaved** with native frames in the final stack trace:
1. Native frame in ART runtime (e.g., `ExecuteMterpImpl`)
2. **Injected Java frame** representing DEX bytecode execution
3. More native frames (e.g., `art::interpreter::Execute`)

This creates a unified view showing both the Java method being executed and the native runtime code executing it.

## Key Implementation Details

### DEX PC Discovery:
The `dex_pc` is discovered during DWARF expression evaluation:
```cpp
// From DwarfSection.cpp
if (is_dex_pc) {
    eval_info->regs_info.regs->set_dex_pc(value);
}
```

### Method Lookup Process:
1. Convert absolute DEX PC to file-relative offset
2. Search cached symbols first
3. If not cached, query ART API: `dex_api_->dex_->FindMethodAtOffset()`
4. Cache result for future lookups
5. Return qualified method name and offset within method

### Thread Safety:
Java unwinding requires additional synchronization because:
- The ART DEX API is not thread-safe
- Multiple processes may map the same DEX file
- Symbol cache needs protection across threads

## Summary

Java stack unwinding in libunwindstack is fundamentally different from native unwinding because it operates on **virtual bytecode execution** rather than native machine instructions. The key innovation is the **virtual frame injection** mechanism that creates artificial stack frames representing Java method execution, which are then interleaved with native runtime frames to provide a complete execution context spanning both Java and native code layers.

This hybrid approach enables unified stack traces that show the complete execution path from Java application code through the Android Runtime (ART) down to native system calls.