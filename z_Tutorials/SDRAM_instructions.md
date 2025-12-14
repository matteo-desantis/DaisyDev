# Using SDRAM for Large DSP Buffers

This document explains how to place large static buffers in the external SDRAM (64 MB) on an STM32H7-based board so that your DSP buffers are not allocated in internal SRAM.

## Summary

- Add an SDRAM region and a named section (e.g. `.sdram_data`) to your linker script.
- Annotate large buffer variables so the compiler places them in that section using `__attribute__((section(".sdram_data")))`.
- Ensure SDRAM is initialized before any access (usually by the board support code / startup routine).
- Rebuild and verify that the SDRAM section is used.

## 1) Linker script changes

Locate your project's linker script (common names: `stm32h7xx_flash.ld`, `STM32H7xxxx_flash.ld`, or similar). Add a memory region for SDRAM and a section that maps to it.


# Using SDRAM for Large DSP Buffers — Summary and Guidance

This document contains a practical summary of recommended approaches for placing large DSP buffers in SDRAM, with compact code examples so you can choose and implement the best fit for your project. The full, implementable `sdram_arena` example follows below under "Answer 3: To Implement".

Summary of approaches (with code snippets)

Approach A — `DSY_SDRAM_BSS` globals passed into instances (explicit per-buffer globals)
- What: Define named SDRAM-backed arrays in a single translation unit and pass them into each object via an `init(buffer, size)` API. This is the pattern used by `patch_sm/Looper`.
- When to use: When buffer sizes and counts are known at compile time and you want deterministic addresses and simple code.

Example (buffers.cpp — one TU):
```cpp
#include "sdram.h" // defines DSY_SDRAM_BSS
constexpr std::size_t PREDELAY_SIZE = 48000;
constexpr std::size_t TANK_SIZE     = 131072;

float DSY_SDRAM_BSS predelay_buf[PREDELAY_SIZE];
float DSY_SDRAM_BSS tank_buf1[TANK_SIZE];
// ... other named buffers
```

Usage in your object initialization:
```cpp
// after board_init() and SDRAM init
mPredelay.init(predelay_buf, PREDELAY_SIZE);
mTankDelay1.init(tank_buf1, TANK_SIZE);
```

Pros: explicit, easy to debug, no allocator complexity.
Cons: manual bookkeeping for many buffers; arrays must live in a single TU.

Approach B — Static template member placed in SDRAM (shared per-template storage)
- What: Make the buffer a `static` member of the template and define it at namespace scope with `DSY_SDRAM_BSS` so the linker places it in `.sdram_bss`.
- When to use: Only when sharing the same backing storage across every instance of a template specialization is acceptable (rare for delay lines).

Example:
```cpp
template <std::size_t N>
class RingBuffer {
public:
        // ...
        static std::array<float, N> mBufferStorage; // declaration only
        float* mRingBufferPtr; // will point at storage or external buffer
};

// Put this in a single .cpp or at the bottom of the header (out-of-class definition):
template <std::size_t N>
std::array<float, N> DSY_SDRAM_BSS dspLib::RingBuffer<N>::mBufferStorage;
```

Pros: easy to define.
Cons: storage is shared across all instances of `RingBuffer<N>` — leads to data corruption if you need independent buffers.

### How `DSY_SDRAM_BSS` works (macro expansion and under-the-hood)

`DSY_SDRAM_BSS` is a convenience macro that expands to a compiler section attribute. In `libDaisy/src/dev/sdram.h` it is defined roughly as:

```cpp
#define DSY_SDRAM_BSS __attribute__((section(".sdram_bss")))
```

What this does:

- At compile time the compiler associates the emitted object symbol with the named section `.sdram_bss` instead of the default `.bss` or `.data` sections.
- At link time the linker script maps `.sdram_bss` to the physical SDRAM region (for example `> SDRAM`), so the symbol receives an address in SDRAM.

Key points to understand and watch for:

- Placement vs initialization: the macro only controls *where* the symbol is placed. It does not configure SDRAM or initialize the controller. You must ensure the SDRAM controller is initialized (board init) before reading/writing these addresses.
- `.sdram_bss` vs `.sdram_data`: use a BSS-style section (typically `NOLOAD`) for large uninitialized buffers to avoid inflating flash image size. If you need initialized contents, map a `.sdram_data` section that is loaded from flash at reset (requires startup copy code or linker `AT` setup).
- Linker symbols: the linker can `PROVIDE` start/end symbols (for example `__sdram_bss_start` / `__sdram_bss_end`) which you can reference from C/C++ (useful for arenas). These are created by the linker script, not the macro.
- Where you define the object matters: putting `DSY_SDRAM_BSS` declarations in headers included by multiple TUs will produce duplicate definitions. Define the array in one `.cpp` or declare it `extern` in headers and define once.
- Not allowed on non-static class members: section attributes require an actual symbol with linkage; non-static class members do not create independent symbols the linker can place into a section, which is why the attribute fails on members. Static members or namespace-scope objects work.
- Symbol visibility and ODR: treat the SDRAM-placed object like any other global — one definition, or use `extern`/`inline` semantics consciously.
- Alignment and DMA: if you will use DMA or require particular alignment, ensure alignment either with `alignas()` or in the linker script.

Example (what the compiler produces conceptually):

Source:
```cpp
float DSY_SDRAM_BSS bigbuf[48000];
```

Conceptually the compiler emits a symbol `bigbuf` and tags it as belonging to `.sdram_bss`; the linker assigns an address in SDRAM for `.sdram_bss` and that symbol points to that SDRAM address at runtime.

In short: the macro is a thin annotation that tags objects for linker placement. The real work that makes them live in SDRAM happens in your linker script and by initializing the SDRAM controller before use.

Approach C — Runtime SDRAM arena allocator (`sdram_alloc`) — recommended when instances need independent storage

- What: Implement a small arena/bump allocator that hands out blocks from `.sdram_bss` at runtime after SDRAM init. Each `RingBuffer`/`DelayLine` calls `sdram_alloc()` in an explicit `init()` to get its own buffer.
- When to use: When you need per-instance buffers but want to avoid declaring many globals; when buffer counts or sizes may vary or be large.

Minimal usage example:
```cpp
// call once after board_init()/SDRAM init
sdram_arena_init();

// inside RingBuffer::init or DelayLine::init (called after sdram init)
mRingBufferPtr = static_cast<float*>(sdram_alloc(sizeof(float) * MaxSamples));
if (!mRingBufferPtr) {
        // handle OOM: fallback, assert, or return error
}
```

Pros: per-instance memory, centralized allocation, flexible for many/dynamic buffers.
Cons: you must ensure SDRAM is initialized before allocating; simple arena needs no-free policy or additional bookkeeping for frees.

Compatibility with templates
- All three approaches work with templated `RingBuffer`/`DelayLine` designs:
    - Approaches A and C: templates accept an external pointer at runtime (`init(buf, size)`) or set `mRingBufferPtr` from `sdram_alloc()`. The template parameter remains a compile-time upper bound and does not force buffer placement.
    - Approach B: template-based static members place storage per specialization and therefore share storage among all instances of that specialization.

When to pick which approach
- Pick Approach A if buffer sizes and counts are fixed and you want explicit, deterministic layout and minimal code changes.
- Pick Approach C (the `sdram_arena`) if you need per-instance buffers, scalability, or want to allocate many buffers without a lot of manual globals. This is the recommended option for `ReverbZ`.
- Pick Approach B only if you intentionally want shared storage per template specialization.

Recommended next step
- Implement the `sdram_arena` (Answer 3) and change `RingBuffer`/`DelayLine` to allocate in an explicit `init()` called after SDRAM initialization. This avoids global constructors touching SDRAM and gives per-instance buffers.

------------------------------------------------

------------------------------------------------
# Answer 3: To Implement
## Linker-script symbols available for an SDRAM arena (checked)

I inspected the project's linker script (`libDaisy/core/STM32H750IB_sram.lds`) and these SDRAM symbols are provided by the script and safe to use from C/C++:

- `__sdram_bss_start`  : start address of the `.sdram_bss` region
- `__sdram_bss_end`    : end address of the `.sdram_bss` region

These are provided in the linker script via `PROVIDE(__sdram_bss_start = _ssdram_bss);` and `PROVIDE(__sdram_bss_end = _esdram_bss);`, so refer to the double-underscore names in your C/C++ code.

### Example `sdram_arena` using the linker symbols

Place this in a single `.cpp`/`.h` pair (e.g. `sdram_arena.h` / `sdram_arena.cpp`) and call `sdram_arena_init()` after your board's SDRAM initialization.

`sdram_arena.h`

```cpp
#pragma once
#include <cstddef>
extern "C" {
    extern char __sdram_bss_start[]; // linker-provided
    extern char __sdram_bss_end[];   // linker-provided
}

void sdram_arena_init(); // optional: set pointer to start
void* sdram_alloc(std::size_t bytes);
void sdram_arena_reset(); // optional: reset allocation pointer
```

`sdram_arena.cpp`

```cpp
#include "sdram_arena.h"
#include <cstdint>
#include <cstring>

static char* sdram_head = nullptr;
static char* sdram_limit = nullptr;

void sdram_arena_init() {
    if(sdram_head == nullptr) {
        sdram_head = __sdram_bss_start;
        sdram_limit = __sdram_bss_end;
    }
}

void* sdram_alloc(std::size_t bytes) {
    if(sdram_head == nullptr) sdram_arena_init();
    // align to 8 bytes
    const std::size_t aligned = (bytes + 7) & ~std::size_t(7);
    if (sdram_head + aligned > sdram_limit) return nullptr;
    void* p = sdram_head;
    sdram_head += aligned;
    std::memset(p, 0, aligned);
    return p;
}

void sdram_arena_reset() {
    sdram_head = __sdram_bss_start;
}
```

Example implementation in RingBuffer.cpp
```cpp
#include "sdram_arena.h" // header above
template<std::size_t MaxSamples>
RingBuffer<MaxSamples>::RingBuffer() {
    // Don't allocate in static ctor: prefer doing it in an explicit init after SDRAM init,
    // or ensure SDRAM is ready before constructing objects that will allocate.
    mRingBufferPtr = nullptr;
    init(); // only initialize indices; real allocation should be explicit
}

template<std::size_t MaxSamples>
void RingBuffer<MaxSamples>::init_with_alloc() {
    // allocate bytes for MaxSamples floats
    void* mem = sdram_alloc(sizeof(float) * MaxSamples);
    if(mem == nullptr) {
        // handle OOM: assert, fallback to heap, or return error code from init
    } else {
        mRingBufferPtr = static_cast<float*>(mem);
    }
    // set indexes and other initialization
}
```
.hpp file to be changed accordingly. 


TODO: Add `sdram_arena.{h,cpp}` to the repo, call `sdram_arena_init()` after `board_init()` (or wherever SDRAM is set up), and update `RingBuffer`/`DelayLine` to optionally allocate their per-instance buffers using `sdram_alloc()` in an explicit `init()` called after board init.

This approach ensures per-instance buffers in SDRAM without sharing static storage, and avoids global constructors touching SDRAM before it is ready.

