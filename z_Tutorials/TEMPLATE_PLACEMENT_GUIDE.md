# Template Definition Placement: `.hpp` vs `.tpp` vs `.cpp`

---

## **Pattern 1: Everything in `.hpp` (Header-Only)**

**When to use:** Most common for templates, embedded systems, libraries you distribute as headers.

```cpp
// RingBuffer.hpp
#ifndef RingBuffer_hpp
#define RingBuffer_hpp

#include <array>

namespace dspLib {

template<std::size_t MaxSamples>
class RingBuffer {
public:
    RingBuffer();  // declaration
    ~RingBuffer(); // declaration
    void reset();  // declaration
    
private:
    std::array<double, MaxSamples> mBufferStorage;
};

// ← DEFINITIONS INLINE IN SAME FILE
template<std::size_t MaxSamples>
RingBuffer<MaxSamples>::RingBuffer() {
    mBufferStorage.fill(0.0);
}

template<std::size_t MaxSamples>
RingBuffer<MaxSamples>::~RingBuffer() {
    // nothing needed - std::array auto-destructs
}

template<std::size_t MaxSamples>
void RingBuffer<MaxSamples>::reset() {
    mBufferStorage.fill(0.0);
}

}  // namespace dspLib
#endif
```

**Pros:**
- ✅ Single file = easier distribution & reuse across projects
- ✅ Compiler sees all definitions when instantiating template (no linker errors)
- ✅ Compiler can optimize aggressively (inline, dead-code elimination)
- ✅ Works universally with all C++ standards (C++98 onwards)

**Cons:**
- ❌ Large headers = slower compilation (every `.cpp` that includes the header recompiles the template code)
- ❌ Reveals implementation details (closed-source libraries must expose code)
- ❌ No separate binary library (you distribute source)
- ❌ Code bloat if template is instantiated many times with different parameters

**Best for:**
- Your dspLib (small, embedded, multi-project reuse)
- Third-party libraries (e.g., STL, Boost)
- Prototyping

---

## **Pattern 2: Separate `.tpp` (Template Implementation)**

**When to use:** Keep the header "clean" while still making definitions visible during compilation.

```cpp
// RingBuffer.hpp
#ifndef RingBuffer_hpp
#define RingBuffer_hpp

#include <array>

namespace dspLib {

template<std::size_t MaxSamples>
class RingBuffer {
public:
    RingBuffer();
    ~RingBuffer();
    void reset();
    
private:
    std::array<double, MaxSamples> mBufferStorage;
};

}  // namespace dspLib

// ← INCLUDE IMPLEMENTATION FILE AT END
#include "RingBuffer.tpp"

#endif
```

```cpp
// RingBuffer.tpp (implementation file, NOT included elsewhere)
// Do NOT add #ifndef guards — this file is ONLY included by RingBuffer.hpp

template<std::size_t MaxSamples>
dspLib::RingBuffer<MaxSamples>::RingBuffer() {
    mBufferStorage.fill(0.0);
}

template<std::size_t MaxSamples>
dspLib::RingBuffer<MaxSamples>::~RingBuffer() {
}

template<std::size_t MaxSamples>
void dspLib::RingBuffer<MaxSamples>::reset() {
    mBufferStorage.fill(0.0);
}
```

**Pros:**
- ✅ Cleaner header file (declarations at top, included impl file at bottom)
- ✅ Still compiles correctly (compiler sees all definitions)
- ✅ Easier to read header vs implementation separately
- ✅ Still header-only distribution (both files shipped together)

**Cons:**
- ❌ Slightly confusing for beginners (`.tpp` is non-standard)
- ❌ Same compilation speed as Pattern 1 (definitions still visible)
- ❌ Same code bloat issues
- ❌ Requires disciplined include at end of `.hpp` (easy to forget)

**Best for:**
- Large template classes where you want header/impl separation for readability
- Professional libraries that distribute headers + docs
- Your dspLib if you want cleaner code organization

---

## **Pattern 3: Separate `.cpp` + Explicit Instantiation**

**When to use:** You only need template in a specific configuration(s), want to compile just once.

```cpp
// RingBuffer.hpp
#ifndef RingBuffer_hpp
#define RingBuffer_hpp

#include <array>

namespace dspLib {

template<std::size_t MaxSamples>
class RingBuffer {
public:
    RingBuffer();
    ~RingBuffer();
    void reset();
    
private:
    std::array<double, MaxSamples> mBufferStorage;
};

// Tell compiler: don't instantiate this template — I'll do it manually in a .cpp
extern template class RingBuffer<48000>;
extern template class RingBuffer<32000>;

}  // namespace dspLib
#endif
```

```cpp
// RingBuffer.cpp - COMPILED ONCE, linked by all users
#include "RingBuffer.hpp"

// Explicit instantiation: compile these specific sizes
template class dspLib::RingBuffer<48000>;
template class dspLib::RingBuffer<32000>;

// Put implementations here (ONLY for explicit instantiation)
template<std::size_t MaxSamples>
dspLib::RingBuffer<MaxSamples>::RingBuffer() {
    mBufferStorage.fill(0.0);
}

template<std::size_t MaxSamples>
dspLib::RingBuffer<MaxSamples>::~RingBuffer() {
}

template<std::size_t MaxSamples>
void dspLib::RingBuffer<MaxSamples>::reset() {
    mBufferStorage.fill(0.0);
}
```

**Pros:**
- ✅ Compiled only once (one binary, linked by all projects)
- ✅ Reduces code bloat (no duplicate instantiations)
- ✅ Fast recompilation of projects (no template recompilation)
- ✅ Can treat as a static library (`.a` / `.lib`)

**Cons:**
- ❌ Limited flexibility: only sizes you explicitly instantiate work (all projects must agree on `MaxSamples`)
- ❌ Linker errors if you try to use a size not explicitly instantiated
- ❌ More setup complexity (extern template, .cpp, explicit list)
- ❌ Not suitable for embedded reuse with varied sizes

**Best for:**
- Production code where you control all template instantiations
- Reducing binary size when size choices are fixed
- NOT ideal for your multi-project dspLib (you want each project to pick its own size)

---

## **Pattern 4: Header-Only + Optional Explicit Instantiation (Hybrid)**

**When to use:** Distribute as header-only but optionally use explicit instantiation in performance-critical projects.

```cpp
// RingBuffer.hpp (header-only by default)
#ifndef RingBuffer_hpp
#define RingBuffer_hpp

#include <array>

namespace dspLib {

template<std::size_t MaxSamples>
class RingBuffer {
public:
    RingBuffer();
    ~RingBuffer();
    void reset();
    
private:
    std::array<double, MaxSamples> mBufferStorage;
};

// Inline or .tpp implementations here
template<std::size_t MaxSamples>
RingBuffer<MaxSamples>::RingBuffer() { /* ... */ }

#ifdef DSPLIB_EXPLICIT_INSTANTIATION
extern template class RingBuffer<48000>;
extern template class RingBuffer<32000>;
#endif

}  // namespace dspLib
#endif
```

```cpp
// Optional: dspLib/RingBuffer_explicit_inst.cpp (only if DSPLIB_EXPLICIT_INSTANTIATION defined)
#include "RingBuffer.hpp"

template class dspLib::RingBuffer<48000>;
template class dspLib::RingBuffer<32000>;
```

**Pros:**
- ✅ Default: header-only (flexibility, any size works)
- ✅ Optional: explicit instantiation for code-size optimization
- ✅ Best of both worlds

**Cons:**
- ❌ More complex setup
- ❌ Requires build-system coordination (define `DSPLIB_EXPLICIT_INSTANTIATION`)

**Best for:**
- Professional libraries targeting diverse users
- Your dspLib if you want future optimization options

---

## **Decision Matrix for Your dspLib**

| Aspect | Your Need | Recommendation |
|--------|-----------|-----------------|
| Multi-project reuse? | YES (different `MaxSamples` per project) | **Pattern 1 (header-only)** or **Pattern 2 (with .tpp)** |
| Embedded systems? | YES (small code size matters) | **Pattern 1 or 2** initially; **Pattern 4** later if bloat occurs |
| Should each project pick size? | YES | Avoid **Pattern 3** (too rigid) |
| Compilation speed important? | MAYBE (Daisy is small) | **Pattern 1/2** is fine; Pattern 3 if build gets slow |
| Source distribution? | YES (dspLib is in your repo) | **Pattern 1 or 2** (definitions visible anyway) |

### **My Recommendation for Your dspLib**
Start with **Pattern 2 (header-only with `.tpp`)**:
- Keep `dspLib/RingBuffer.hpp` with declarations
- Add `dspLib/RingBuffer.tpp` with implementations
- Include `.tpp` at end of `.hpp`
- Same benefits as Pattern 1, but cleaner code organization
- Easy to switch to Pattern 4 later if binary bloat becomes a problem

---

## **Why NOT Use `.cpp` Directly?**

```cpp
// ❌ WRONG - DO NOT DO THIS
// RingBuffer.cpp
template<std::size_t MaxSamples>
RingBuffer<MaxSamples>::RingBuffer() { }
```

When you `#include "RingBuffer.hpp"` in a project `.cpp`, the compiler won't see the template definitions in `RingBuffer.cpp` (it's compiled separately). Result: **linker error** (undefined reference).

The compiler needs template definitions **visible during instantiation**. That means:
1. In the same header (Pattern 1)
2. In a `.tpp` included by the header (Pattern 2)
3. In a `.cpp` but with `extern template` hints (Pattern 3 — advanced)

---

## **Quick Summary Table**

```
┌─────────────────────────────────────────────────────────────────┐
│ Placement                    │ Visibility      │ Compile Speed   │
├─────────────────────────────────────────────────────────────────┤
│ .hpp (inline)                │ ✅ Always       │ ⏱ Slower        │
│ .tpp (included by .hpp)      │ ✅ Always       │ ⏱ Slower        │
│ .cpp (separate)              │ ❌ Never*       │ ✅ Faster        │
│ .cpp (explicit instantiation)│ ✅ Limited      │ ✅ Fastest       │
└─────────────────────────────────────────────────────────────────┘
*unless using extern template / explicit instantiation (advanced)
```

---

## **For Your ReverbZ Templating**

I'd suggest:

```
dspLib/
  RingBuffer.hpp (declares template class)
  RingBuffer.tpp (includes implementations)
  DelayLine.hpp (declares template class)
  DelayLine.tpp (includes implementations)
  LFO.hpp
  LFO.tpp
  AllPass.hpp
  AllPass.tpp

OwnProjects/_projLib/
  ReverbZ.hpp (declares template<std::size_t MaxSamples> class ReverbZ)
  ReverbZ.tpp (includes implementations)
```

Each `.tpp` file is only included by its corresponding `.hpp` at the very end. That way:
- Headers are readable (declarations at top)
- Implementations are organized (separate `.tpp` files)
- Compilation works (definitions visible when needed)
- No linker errors
- Embeds naturally in your distributed repo

---

## **Key Takeaway**

For your embedded multi-project dspLib, **use Pattern 2 (header-only with `.tpp`)**. It's:
- ✅ Flexible (each project picks its own `MaxSamples`)
- ✅ Safe (no linker errors)
- ✅ Organized (clean header/implementation separation)
- ✅ Familiar (similar to STL practices)
- ✅ Futureproof (can optimize with Pattern 4 if needed)
