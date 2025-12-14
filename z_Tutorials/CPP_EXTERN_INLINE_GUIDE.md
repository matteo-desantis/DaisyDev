# C++ `extern` and `inline` Syntax Guide

This guide explains the `extern`, `extern "C"`, and `inline` keywords commonly used in embedded C++ projects, particularly when interfacing with linker scripts and managing symbol linkage.

## Table of Contents
1. [extern "C" - C Linkage](#extern-c---c-linkage)
2. [extern Keyword](#extern-keyword)
3. [Array Declaration with Empty Brackets](#array-declaration-with-empty-brackets)
4. [inline Functions](#inline-functions)
5. [inline Variables (C++17)](#inline-variables-c17)
6. [Practical Example: sdramArena](#practical-example-sdramarena)
7. [Summary Table](#summary-table)

---

## `extern "C"` - C Linkage

### What It Does

`extern "C"` tells the C++ compiler to use **C-style linkage** for the enclosed declarations instead of C++ linkage. This prevents **name mangling**.

### Syntax

```cpp
extern "C" {
    void myFunction(int x);
    int myVariable;
}

// Or for a single declaration:
extern "C" void myFunction(int x);
```

### Name Mangling Explained

C++ supports function overloading, so the compiler encodes type information into symbol names:

```cpp
// C++ (without extern "C")
void print(int x);        // mangled to something like "_Z5printi"
void print(double x);     // mangled to something like "_Z5printd"

// C or extern "C"
extern "C" void print(int x);  // stays as "print"
// Note: cannot overload extern "C" functions!
```

### When to Use `extern "C"`

✅ **Use when:**
- Calling C code from C++
- Being called by C code from C++
- Accessing linker-provided symbols (from linker scripts)
- Interfacing with assembly code
- Creating plugin APIs that must work across compilers

❌ **Don't use when:**
- Working entirely in C++
- Using C++ features (overloading, templates, namespaces)

### Example: Linker Script Symbols

```cpp
extern "C" {
    // Linker script provides these symbols
    extern char __sdram_bss_start[];
    extern char __sdram_bss_end[];
}

void initSDRAM() {
    char* start = __sdram_bss_start;  // Use the linker symbol
    char* end = __sdram_bss_end;
    size_t size = end - start;
}
```

---

## `extern` Keyword

### What It Does

`extern` declares that a symbol (variable or function) is **defined elsewhere** — in another translation unit (.cpp file) or by the linker.

### Declaration vs Definition

```cpp
// Declaration (extern) - no storage allocated
extern int globalCounter;

// Definition - storage allocated, optionally initialized
int globalCounter = 0;
```

### Usage Pattern

**header.h:**
```cpp
#ifndef HEADER_H
#define HEADER_H

extern int sharedValue;     // Declaration
void incrementValue();      // Function declarations are implicitly extern

#endif
```

**source1.cpp:**
```cpp
#include "header.h"

int sharedValue = 100;  // Definition (one per program)

void incrementValue() {
    sharedValue++;
}
```

**source2.cpp:**
```cpp
#include "header.h"

void useValue() {
    incrementValue();           // Uses extern function
    int x = sharedValue + 10;  // Uses extern variable
}
```

### Key Points

- `extern` = "I promise this exists somewhere, find it at link time"
- Variables can be declared `extern` many times but defined **only once** (One Definition Rule)
- Functions are implicitly `extern` (you rarely write it explicitly)
- Const variables are implicitly internal linkage unless declared `extern`

### Special Case: `extern const`

```cpp
// header.h
extern const int MAX_SIZE;  // Must use extern for const to share across files

// source.cpp
const int MAX_SIZE = 1024;  // Definition
```

---

## Array Declaration with Empty Brackets

### What It Means

```cpp
extern char myArray[];
```

Declares an array of **unknown size** — the size is determined elsewhere (at definition or by the linker).

### Common Use: Linker Symbols

Linker scripts define symbols that mark memory addresses. These are declared as arrays:

```cpp
extern "C" {
    extern char __sdram_bss_start[];
    extern char __sdram_bss_end[];
}
```

**Why arrays, not pointers?**

- The symbol **IS** the address itself (not a pointer variable that contains an address)
- Taking the address with `&symbol` or using it directly as `symbol` gives you the memory location
- This is a historical C convention for linker symbols

### Usage

```cpp
// Treat as pointers to get addresses
char* start_ptr = __sdram_bss_start;
char* end_ptr = __sdram_bss_end;

// Calculate region size
size_t sdram_size = end_ptr - start_ptr;

// Or use address-of (equivalent)
void* start_addr = &__sdram_bss_start[0];
```

### Linker Script Definition

In the linker script (`.lds` file):
```ld
.sdram_bss (NOLOAD) :
{
    . = ALIGN(4);
    _ssdram_bss = .;                      /* Internal symbol */
    PROVIDE(__sdram_bss_start = _ssdram_bss);  /* Exported symbol */
    *(.sdram_bss)
    *(.sdram_bss*)
    _esdram_bss = .;
    PROVIDE(__sdram_bss_end = _esdram_bss);
} > SDRAM
```

The `PROVIDE()` directive makes symbols available to C/C++ code.

---

## `inline` Functions

### What It Does

Suggests to the compiler to **insert the function body** directly at each call site instead of making a function call.

### Benefits

- **Eliminates call overhead** (no stack frame setup, no jump)
- **Better optimization** (compiler can optimize across inlined code)
- **Faster for small, frequently-called functions**

### Syntax

```cpp
inline int square(int x) {
    return x * x;
}

// Usage
int result = square(5);  // May be expanded to: int result = 5 * 5;
```

### Modern C++ (C++11+)

The `inline` keyword has evolved:

**Traditional meaning:**
- Suggestion to inline the code (compiler may ignore)

**Modern meaning (also):**
- Allows multiple definitions across translation units (One Definition Rule exemption)
- Function can be defined in a header without causing linker errors

### Header-Only Functions

```cpp
// myheader.h
inline int add(int a, int b) {
    return a + b;
}
```

This can be included in multiple `.cpp` files without linker errors. The compiler ensures only one instance exists in the final binary.

### When to Use

✅ **Good candidates for inline:**
- Small functions (1-5 lines)
- Frequently called functions (hot paths)
- Accessors/getters: `inline float getValue() const { return value; }`
- Simple math: `inline float lerp(float a, float b, float t) { return a + t * (b - a); }`

❌ **Avoid inlining:**
- Large functions (increases code size)
- Recursive functions
- Functions with loops (unless very short)
- Virtual functions (can't be inlined in most cases)

### Compiler Override

```cpp
// Force inline (compiler-specific)
__attribute__((always_inline)) inline void criticalFunction() { /* ... */ }

// Prevent inline (compiler-specific)
__attribute__((noinline)) void debugFunction() { /* ... */ }
```

---

## `inline` Variables (C++17)

C++17 introduced `inline` for variables, allowing them to be defined in headers.

### Problem Without `inline`

**header.h:**
```cpp
// This will cause linker errors if included in multiple .cpp files!
int globalCounter = 0;  // Multiple definitions
```

### Solution with `inline`

**header.h:**
```cpp
inline int globalCounter = 0;  // OK: one instance across all TUs
```

### Use Cases

```cpp
// Constants in headers
inline constexpr double PI = 3.14159265359;

// Static member variables
class MyClass {
    inline static int instanceCount = 0;  // C++17
};

// Global configuration
inline std::string configPath = "/etc/myapp/config";
```

### Before C++17

You had to use workarounds:

```cpp
// Workaround 1: extern + separate definition
// header.h
extern int globalCounter;
// source.cpp
int globalCounter = 0;

// Workaround 2: function-local static
inline int& getCounter() {
    static int counter = 0;
    return counter;
}
```

---

## Practical Example: sdramArena

Here's the complete pattern from `dspLib/Utils/sdramArena.cpp`:

```cpp
#include "sdramArena.h"
#include <cstdint>
#include <cstring>

// Linker-provided symbols (from STM32H750IB_sram.lds)
extern "C" {
    extern char __sdram_bss_start[];  // Start address of SDRAM BSS section
    extern char __sdram_bss_end[];    // End address of SDRAM BSS section
}

// Internal state (file-scope, not exposed in header)
static char* sdram_head = nullptr;
static char* sdram_limit = nullptr;

void sdramArenaInit() {
    if(sdram_head == nullptr) {
        // Convert linker symbols to pointers
        sdram_head = __sdram_bss_start;
        sdram_limit = __sdram_bss_end;
    }
}

void* sdramArenaAlloc(std::size_t bytes) {
    if(sdram_head == nullptr) sdramArenaInit();
    
    // Align to 8 bytes
    const std::size_t aligned = (bytes + 7) & ~std::size_t(7);
    
    // Check if enough space
    if (sdram_head + aligned > sdram_limit) return nullptr;
    
    // Allocate and advance pointer
    void* p = sdram_head;
    sdram_head += aligned;
    
    // Zero the memory
    std::memset(p, 0, aligned);
    
    return p;
}

void sdramArenaReset() {
    sdram_head = __sdram_bss_start;
}
```

**Why each piece is needed:**

1. **`extern "C"`** - Linker script symbols use C naming (no mangling)
2. **`extern char []`** - Declares symbols defined by linker (addresses, not data)
3. **`static` variables** - Internal state, not visible outside this file
4. **Pointer arithmetic** - Treat linker symbols as memory addresses

---

## Summary Table

| Keyword/Syntax | Purpose | Use Case |
|----------------|---------|----------|
| `extern` | Declare symbol defined elsewhere | Share variables/functions across files |
| `extern "C"` | Use C linkage (no name mangling) | Interface with C code, linker scripts, assembly |
| `extern char[]` | Array of unknown size, defined elsewhere | Linker-provided address symbols |
| `inline` (function) | Suggest code insertion at call site | Small, frequently-called functions |
| `inline` (C++17 variable) | Allow definition in header, one instance | Header-only constants, static members |
| `static` (file scope) | Internal linkage (private to file) | Implementation details, file-local state |

---

## Quick Reference: Common Patterns

### Pattern 1: Shared Global Variable

```cpp
// header.h
extern int sharedData;

// source.cpp
int sharedData = 42;
```

### Pattern 2: Linker Script Symbol Access

```cpp
extern "C" {
    extern char __heap_start[];
    extern char __heap_end[];
}

size_t heap_size = __heap_end - __heap_start;
```

### Pattern 3: Header-Only Function

```cpp
// header.h
inline float clamp(float x, float min, float max) {
    return x < min ? min : (x > max ? max : x);
}
```

### Pattern 4: Header-Only Constant (C++17)

```cpp
// header.h
inline constexpr int BUFFER_SIZE = 1024;
```

### Pattern 5: C++ Calling C Library

```cpp
extern "C" {
    #include "c_library.h"  // C header
}

void myFunction() {
    c_library_function();  // No name mangling issues
}
```

---

## Best Practices

1. **Use `extern "C"` for all linker script symbols** to avoid name mangling
2. **Declare `extern` in headers, define in one `.cpp`** for global variables
3. **Use `inline` for small, header-only functions** (C++11+)
4. **Use `inline` for header constants** (C++17+)
5. **Prefer `static` for file-local helpers** (internal linkage)
6. **Let the compiler decide** on function inlining (modern optimizers are smart)
7. **Check generated assembly** to verify inlining happened when critical

---

## Further Reading

- C++ Standard: Linkage and Storage Duration
- Linker Scripts: GNU LD Manual
- Name Mangling: `c++filt` tool to demangle symbols
- Inline Optimization: Compiler optimization flags (`-O2`, `-O3`)
