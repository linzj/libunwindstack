# Key Code for Java Frame Handling in libunwindstack

## 1. Java Frame Identification

### How Java Frames are Identified

Java frames are identified by the presence of a **`dex_pc`** (DEX program counter) value in the CPU registers. This is discovered during DWARF unwinding:

```cpp
// From DwarfSection.cpp:475-476
if (is_dex_pc) {
    eval_info->regs_info.regs->set_dex_pc(value);
}
```

The `dex_pc` is set when DWARF expression evaluation encounters a special DEX PC marker:

```cpp
// From DwarfOp.cpp:1514 
dex_pc_set_ = true;  // When special DWARF operation indicates DEX PC
```

### Java Frame Detection in Main Unwinding Loop

```cpp
// From Unwinder.cpp:193-203
if (regs_->dex_pc() != 0) {
    // Add a frame to represent the dex file.
    FillInDexFrame();
    // Clear the dex pc so that we don't repeat this frame later.
    regs_->set_dex_pc(0);

    // Make sure there is enough room for the real frame.
    if (frames_.size() == max_frames_) {
        last_error_.code = ERROR_MAX_FRAMES_EXCEEDED;
        break;
    }
}
```

**Key Point**: Java frames are **virtual frames** injected when `dex_pc != 0` is detected.

## 2. Java Frame Creation (FillInDexFrame)

### Complete Java Frame Injection Code

```cpp
// From Unwinder.cpp:47-85
// Inject extra 'virtual' frame that represents the dex pc data.
// The dex pc is a magic register defined in the Mterp interpreter,
// and thus it will be restored/observed in the frame after it.
// Adding the dex frame first here will create something like:
//   #7 pc 0015fa20 core.vdex   java.util.Arrays.binarySearch+8
//   #8 pc 006b1ba1 libartd.so  ExecuteMterpImpl+14625
//   #9 pc 0039a1ef libartd.so  art::interpreter::Execute+719
void Unwinder::FillInDexFrame() {
    size_t frame_num = frames_.size();
    frames_.resize(frame_num + 1);
    FrameData* frame = &frames_.at(frame_num);
    frame->num = frame_num;

    uint64_t dex_pc = regs_->dex_pc();
    frame->pc = dex_pc;
    frame->sp = regs_->sp();

    // Find memory mapping for DEX PC
    frame->map_info = maps_->Find(dex_pc);
    if (frame->map_info != nullptr) {
        frame->rel_pc = dex_pc - frame->map_info->start();
        // Initialize the load bias for this map so subsequent calls
        // to GetLoadBias() will always return data.
        frame->map_info->set_load_bias(0);
    } else {
        frame->rel_pc = dex_pc;
        warnings_ |= WARNING_DEX_PC_NOT_IN_MAP;
        return;
    }

    if (!resolve_names_) {
        return;
    }

#if defined(DEXFILE_SUPPORT)
    if (dex_files_ == nullptr) {
        return;
    }

    // CRITICAL: This is where Java method names are resolved
    dex_files_->GetFunctionName(maps_, dex_pc, &frame->function_name, &frame->function_offset);
#endif
}
```

### Frame Structure for Java vs Native

**Java Frame FrameData**:
- `pc` = `dex_pc` (virtual bytecode address)
- `rel_pc` = offset within DEX file
- `map_info` = points to DEX file mapping (.vdex, .dex, .oat)
- `function_name` = Java method name (e.g., "java.util.Arrays.binarySearch")

**Native Frame FrameData**:
- `pc` = native instruction address  
- `rel_pc` = offset within native library
- `map_info` = points to ELF library mapping (.so)
- `function_name` = native function name

## 3. Java VM Search Mechanism

### DEX File Loading and Discovery

```cpp
// From DexFiles.cpp:32-35
template <>
bool GlobalDebugInterface<DexFile>::Load(Maps* maps, std::shared_ptr<Memory>& memory, uint64_t addr,
                                         uint64_t size, /*out*/ std::shared_ptr<DexFile>& dex) {
    dex = DexFile::Create(addr, size, memory.get(), maps->Find(addr).get());
    return dex.get() != nullptr;
}
```

### Java VM Integration via JIT Debug Interface

The key mechanism for finding Java symbols is through the **JIT Debug Interface** which connects to the Android Runtime (ART):

```cpp
// From DexFiles.cpp:37-40
std::unique_ptr<DexFiles> CreateDexFiles(ArchEnum arch, std::shared_ptr<Memory>& memory,
                                         std::vector<std::string> search_libs) {
    return CreateGlobalDebugImpl<DexFile>(arch, memory, search_libs, "__dex_debug_descriptor");
}
```

**Key Symbol**: `__dex_debug_descriptor` - This is the global variable that ART exposes to allow external tools to find DEX files.

### GlobalDebugImpl - The Java VM Search Engine

```cpp
// From GlobalDebugImpl.h:169-172
bool GetFunctionName(Maps* maps, uint64_t pc, SharedString* name, uint64_t* offset) {
    // NB: If symfiles overlap in PC ranges, this will check all of them.
    return ForEachSymfile(maps, pc, [pc, name, offset](Symfile* file) {
        return file->GetFunctionName(pc, name, offset);
    });
}
```

### JIT Interface Structure for DEX Discovery

```cpp
// From GlobalDebugImpl.h:49-60
struct JITCodeEntry {
    Uintptr_T next;           // Linked list of entries
    Uintptr_T prev;
    Uintptr_T symfile_addr;   // Address of DEX file in memory
    Uint64_T symfile_size;    // Size of DEX file
    // Android-specific fields:
    Uint64_T timestamp;
    uint32_t seqlock;         // For thread-safe access
};

struct JITDescriptor {
    uint32_t version;
    uint32_t action_flag;
    Uintptr_T relevant_entry;
    Uintptr_T first_entry;    // Head of linked list
    // Android-specific fields:
    uint8_t magic[8];         // "Android2"
    uint32_t flags;
    uint32_t sizeof_descriptor;
    uint32_t sizeof_entry;
    uint32_t seqlock;
    Uint64_T timestamp;
};
```

### Complete DEX Symbol Search Process

```cpp
// From GlobalDebugImpl.h:145-165
template <typename Callback>
bool ForEachSymfile(Maps* maps, uint64_t pc, Callback callback) {
    std::lock_guard<std::mutex> guard(lock_);
    if (descriptor_addr_ == 0) {
        // STEP 1: Find the __dex_debug_descriptor in ART
        FindAndReadVariable(maps, global_variable_name_);
        if (descriptor_addr_ == 0) {
            return false;
        }
    }

    // STEP 2: Try to find the entry in already loaded symbol files.
    for (auto& it : entries_) {
        Symfile* symfile = it.second.get();
        // Check seqlock to make sure that entry is still valid
        if (symfile->IsValidPc(pc) && CheckSeqlock(it.first) && callback(symfile)) {
            return true;
        }
    }

    // STEP 3: Update all entries and retry.
    ReadAllEntries(maps);
    for (auto& it : entries_) {
        Symfile* symfile = it.second.get();
        if (symfile->IsValidPc(pc) && callback(symfile)) {
            return true;
        }
    }

    return false;
}
```

## 4. DEX Method Lookup Implementation

### Java Method Resolution

```cpp
// From DexFile.cpp:126-145
bool DexFile::GetFunctionName(uint64_t dex_pc, SharedString* method_name, uint64_t* method_offset) {
    uint64_t dex_offset = dex_pc - base_addr_;  // Convert absolute PC to file-relative offset.

    // Lookup the function in the cache.
    std::lock_guard<std::mutex> guard(dex_api_->lock_);  // Protect both the symbols and the C API.
    auto it = symbols_.upper_bound(dex_offset);
    if (it == symbols_.end() || dex_offset < it->second.offset) {
        // Lookup the function in the underlying dex file using ART API.
        size_t found = dex_api_->dex_->FindMethodAtOffset(dex_offset, [&](const auto& method) {
            size_t code_size, name_size;
            uint32_t offset = method.GetCodeOffset(&code_size);
            const char* name = method.GetQualifiedName(/*with_params=*/false, &name_size);
            it = symbols_.emplace(offset + code_size, Info{offset, std::string(name, name_size)}).first;
        });
        if (found == 0) {
            return false;
        }
    }

    // Return the found function.
    *method_offset = dex_offset - it->second.offset;
    *method_name = it->second.name;
    return true;
}
```

### Critical ART API Call

The most important line for Java symbol resolution:

```cpp
dex_api_->dex_->FindMethodAtOffset(dex_offset, callback)
```

This calls into the Android Runtime's DEX file parser to find the Java method at a specific bytecode offset.

## 5. Key Data Structures

### Register State Management

```cpp
// From include/unwindstack/Regs.h:64-65
uint64_t dex_pc() { return dex_pc_; }
void set_dex_pc(uint64_t dex_pc) { dex_pc_ = dex_pc; }
```

### DEX File Caching

```cpp
// From DexFile.h:78-80
using MappedFileKey = std::tuple<std::string, uint64_t, uint64_t>;  // (path, offset, size)
static std::map<MappedFileKey, std::weak_ptr<DexFileApi>> g_mapped_dex_files;
static std::mutex g_lock;  // Guards the static cache above.
```

## Summary

**Java Frame Handling Pipeline**:

1. **Detection**: DWARF unwinding sets `dex_pc` when encountering Java bytecode frame
2. **Injection**: `FillInDexFrame()` creates virtual frame with `dex_pc` as frame PC  
3. **VM Search**: `GlobalDebugImpl` finds DEX files via `__dex_debug_descriptor` from ART
4. **Symbol Resolution**: `DexFile::GetFunctionName()` uses ART API to resolve Java method names
5. **Caching**: Results cached for performance with thread-safe seqlock mechanism

The entire system creates a seamless integration between native stack unwinding and Java runtime introspection, allowing unified stack traces that span both native and Java execution contexts.