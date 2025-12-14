# SDRAM Allocation Problem - Complete Resolution

## The Problem

Your ReverbZ reverb algorithm requires **large delay line buffers** (multiple instances of `RingBuffer<8192>`, `DelayLine<8192>`, `AllPass<8192>`) that exceeded available **SRAM (512KB)**. The solution was to allocate these buffers in the **external SDRAM (64MB)** on the Daisy Patch SM.

## Initial Issues Encountered

### 1. **Constructor Called During Static Initialization**
**Problem:** The global `ReverbZ_t reverbz(FS_REVERBZ);` object had its constructor call `init()`, which tried to allocate SDRAM buffers **before `main()` started**, before hardware/SDRAM initialization.

**Solution:** Removed `init()` call from `ReverbZ` constructor. Now `init()` must be called explicitly after `patch.Init()` and `sdramArenaInit()`.

**File:** `/Users/matteodesantis/EurorackProgramming/Git/DaisyDev/OwnProjects/_projLib/ReverbZ.tpp`
```cpp
// Constructor no longer calls init()
ReverbZ<MaxSamples>::ReverbZ(int sampleRate) {
    mFs_ = sampleRate;
    // NOTE: init() must be called manually after hardware/SDRAM initialization
}
```

### 2. **Linker Symbols Were NULL**
**Problem:** When stepping through `sdramArenaInit()` in the debugger:
- `__sdram_bss_start = 0x0` (NULL)
- `__sdram_bss_end = 0x0` (NULL)
- This meant **no SDRAM region was available** for allocation

**Root Cause:** The `.sdram_bss` section in the linker script was **empty** (size = 0 bytes) because nothing was actually placed in it.

### 3. **Linker Garbage Collection Removed SDRAM Section**
**Problem:** Even after declaring a large buffer with `DSY_SDRAM_BSS`, the linker's `--gc-sections` flag **removed the entire section** because it was never referenced in the code.

**Evidence from build output:**
```
SDRAM:          0 GB        64 MB      0.00%
```

**Linker map showed:**
```
.sdram_bss     0x0000000000000000  0x1000000 build/ReverbZpatch.o  // 16MB in object file
.sdram_bss     0x00000000c0000000        0x0                        // 0 bytes in final ELF
```

## The Solution

### Step 1: Reserve SDRAM Space with Static Buffer

Added a **16MB static buffer** marked with `DSY_SDRAM_BSS` attribute to force the linker to allocate space in the `.sdram_bss` section:

**File:** `/Users/matteodesantis/EurorackProgramming/Git/DaisyDev/OwnProjects/ReverbZpatch/ReverbZpatch.cpp`
```cpp
#include "dev/sdram.h"  // For DSY_SDRAM_BSS macro

// Reserve SDRAM space for arena allocator
#define SDRAM_BUFFER_SIZE (16 * 1024 * 1024)  // 16MB reserve
volatile uint8_t DSY_SDRAM_BSS __attribute__((used)) sdram_pool[SDRAM_BUFFER_SIZE];
```

**Key details:**
- `DSY_SDRAM_BSS` expands to `__attribute__((section(".sdram_bss")))`
- `__attribute__((used))` tells the linker not to garbage-collect
- `volatile` prevents compiler optimization
- Requires `#include "dev/sdram.h"` for the macro definition

### Step 2: Reference the Buffer to Prevent Garbage Collection

Even with `__attribute__((used))`, the linker still removed the section. The critical fix was to **actually reference the variable** in code:

**File:** `/Users/matteodesantis/EurorackProgramming/Git/DaisyDev/OwnProjects/ReverbZpatch/ReverbZpatch.cpp`
```cpp
int main(void) {
    patch.Init();
    button.Init(patch.B7);
    toggle.Init(patch.B8);
    
    // Touch sdram_pool to prevent linker garbage collection
    sdram_pool[0] = 0;  // ← THIS IS CRITICAL
    
    System::Delay(100);
    sdramArenaInit();
    System::Delay(100);
    reverbz.init();
    // ... rest of main
}
```

This simple assignment ensures the linker **must keep** the entire `sdram_pool` buffer.

## Final Result

### Build Output (Success)
```
Memory region         Used Size  Region Size  %age Used
           FLASH:       88132 B       128 KB     67.24%
            SRAM:       14740 B       512 KB      2.81%
           SDRAM:         16 MB        64 MB     25.00%  ← SUCCESS!
```

### Linker Symbols (Verified)
```bash
$ arm-none-eabi-nm ReverbZpatch.elf | grep sdram
c0000000 B __sdram_bss_start  # Start of 64MB SDRAM at 0xC0000000
c1000000 B __sdram_bss_end    # +16MB = 0xC1000000
c0000000 B sdram_pool          # 16MB buffer starts here
```

### Runtime Behavior
1. **`patch.Init()`** initializes hardware including SDRAM controller
2. **`sdram_pool[0] = 0`** ensures buffer is linked
3. **`sdramArenaInit()`** sets up allocator with valid pointers:
   - `sdram_head = 0xC0000000`
   - `sdram_limit = 0xC1000000`
4. **`reverbz.init()`** allocates all delay line buffers from the 16MB pool via `sdramArenaAlloc()`

## Debug Build Configuration

Also configured debug builds to use `-O0` for better debugging visibility:

**File:** `/Users/matteodesantis/EurorackProgramming/Git/DaisyDev/OwnProjects/ReverbZpatch/Makefile`
```makefile
BUILD_TYPE ?= release

ifeq ($(BUILD_TYPE),debug)
    OPT = -O0 -g3
else
    OPT = -O2
endif
```

**File:** `/Users/matteodesantis/EurorackProgramming/Git/DaisyDev/dspLib/Makefile`
```makefile
BUILD_TYPE ?= release

ifeq ($(BUILD_TYPE),debug)
    OPT = -O0 -g3
else
    OPT = -O2
endif
```

**Usage:**
```bash
# Debug build (no optimization, all variables visible)
BUILD_TYPE=debug make all

# Release build (optimized)
BUILD_TYPE=release make all
```

## Key Takeaways

1. **NOLOAD sections with `--gc-sections`** require variables to be **actually referenced** in code, not just marked `__attribute__((used))`

2. **Global object constructors run before `main()`** - never allocate hardware resources (SDRAM, peripherals) in constructors

3. **Linker symbols** (`__sdram_bss_start`/`end`) are only valid if something is actually placed in the section

4. **Always verify with tools:**
   - `arm-none-eabi-nm` to check symbol addresses
   - `arm-none-eabi-objdump -h` to verify section sizes
   - Linker map file (`*.map`) to trace section placement

5. **Touch/reference dummy buffers** used for memory reservation to prevent garbage collection

## Files Modified

1. **`OwnProjects/_projLib/ReverbZ.tpp`** - Removed `init()` from constructor
2. **`OwnProjects/ReverbZpatch/ReverbZpatch.cpp`** - Added SDRAM pool reservation and reference
3. **`OwnProjects/ReverbZpatch/Makefile`** - Added `BUILD_TYPE` support for debug builds
4. **`dspLib/Makefile`** - Added `BUILD_TYPE` support for debug builds

## Troubleshooting Tips

### Problem: SDRAM still shows 0 bytes used

**Check 1: Verify DSY_SDRAM_BSS macro is defined**
```bash
arm-none-eabi-g++ -E your_file.cpp -I../../libDaisy -I../../libDaisy/src | grep DSY_SDRAM_BSS
```
Should show `__attribute__((section(".sdram_bss")))`, not the raw macro name.

**Check 2: Verify section exists in object file**
```bash
arm-none-eabi-objdump -h build/YourFile.o | grep sdram
```
Should show `.sdram_bss` with non-zero size.

**Check 3: Verify section exists in final ELF**
```bash
arm-none-eabi-objdump -h build/YourProject.elf | grep sdram
```
If size is 0, the section was garbage collected.

**Check 4: Verify linker symbols**
```bash
arm-none-eabi-nm build/YourProject.elf | grep sdram
```
If `__sdram_bss_start == __sdram_bss_end`, no space was allocated.

**Fix:** Ensure you **reference the SDRAM buffer** somewhere in your code (e.g., `sdram_pool[0] = 0;` in `main()`).

### Problem: Variables still show as `<optimized out>` in debugger

**Solution:** Build all libraries and your project with `-O0`:
```bash
# Clean everything
make -C /path/to/libDaisy clean
make -C /path/to/DaisySP clean
make -C /path/to/dspLib clean
make -C /path/to/YourProject clean

# Rebuild with -O0
OPT='-O0 -g3' make -C /path/to/libDaisy
OPT='-O0 -g3' make -C /path/to/DaisySP
BUILD_TYPE=debug make -C /path/to/dspLib
BUILD_TYPE=debug make -C /path/to/YourProject
```

Or use the `BUILD_TYPE` variable for projects that support it:
```bash
BUILD_TYPE=debug make all
```

### Problem: Hard fault or crash at runtime

**Common causes:**
1. **SDRAM not initialized before use** - ensure `patch.Init()` is called first
2. **Allocating before `sdramArenaInit()`** - call `sdramArenaInit()` before any `init()` methods that allocate
3. **Out of SDRAM** - increase `SDRAM_BUFFER_SIZE` or reduce buffer allocations

**Debug sequence:**
```cpp
patch.Init();              // Initialize hardware (including SDRAM)
System::Delay(100);        // Let hardware stabilize
sdramArenaInit();          // Set up arena allocator
System::Delay(100);        // Safety delay
yourObject.init();         // Now safe to allocate from SDRAM
```

## Summary

The system now successfully allocates ~16MB of SDRAM for reverb buffers, with the arena allocator providing dynamic allocation within that pool. The key insight was that linker garbage collection requires **actual code references** to sections, not just attributes or declarations.
