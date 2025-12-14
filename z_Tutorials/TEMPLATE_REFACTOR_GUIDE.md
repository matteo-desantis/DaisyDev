# RingBuffer Template Refactoring Guide

## Overview
Convert `RingBuffer` and its derived classes (`DelayLine`, `LFO`, `AllPass`) from dynamic allocation to compile-time template-based static allocation. This eliminates runtime memory allocation issues and improves predictability for embedded systems.

---

## Architecture

### Current Structure (Dynamic)
```
RingBuffer (base)
  ├─ DelayLine (extends RingBuffer)
  │  └─ Uses LFO internally
  ├─ AllPass (extends DelayLine → extends RingBuffer)
  └─ LFO (extends RingBuffer)
```

### New Structure (Template-based)
```
RingBuffer<MaxSamples> (template base)
  ├─ DelayLine<MaxSamples> (template, extends RingBuffer<MaxSamples>)
  │  └─ Uses LFO<MaxSamples> internally
  ├─ AllPass<MaxSamples> (template, extends DelayLine<MaxSamples>)
  └─ LFO<MaxSamples> (template, extends RingBuffer<MaxSamples>)
```

---

## Step-by-Step Implementation

### Step 1: Define Buffer Size Constant in ReverbZpatch -> Done

**File**: `OwnProjects/ReverbZpatch/ReverbZpatch.hpp` (create new header)

Create a project-level configuration header with the buffer size constant:

```cpp
#ifndef REVERBZPATCH_CONFIG_HPP
#define REVERBZPATCH_CONFIG_HPP

// Define the maximum buffer size for all dspLib templates
// This is used by RingBuffer, DelayLine, LFO, and AllPass classes
// Value example: 48000 samples @ 48kHz = 1 second of audio
// Use std::size_t for the template parameter type
constexpr std::size_t DSPLIB_MAX_BUFFER_SIZE = 48000;

#endif // REVERBZPATCH_CONFIG_HPP
```
[ -> Step 1: done 17-11-2025]

### Step 2: Refactor RingBuffer to Template

**File**: `dspLib/RingBuffer.hpp`

Replace the current class with a template version:

```cpp
#ifndef RingBuffer_hpp
#define RingBuffer_hpp

#include <cstring>
#include <array>

namespace dspLib {

template<std::size_t MaxSamples>
class RingBuffer
{
public:
    RingBuffer();                    // constructor
    ~RingBuffer();                   // destructor

    int mRingBufferLength;                  // Ring buffer size (actual used, ≤ MaxSamples)
    int mReadIdx, mWriteIdx;                // read/write indexes
    double* mRingBufferPtr;                 // pointer to buffer data
    
    void reset();
    void updateReadIndex();
    void updateWriteIndex();

protected:
    // Static storage for buffer (allocated at compile-time)
    std::array<double, MaxSamples> mBufferStorage;
    
    void commonConstructor();
    int updateRingBufferIdx(int index);
};

// Implementation in RingBuffer.cpp (will be template specialization or inline)
template<std::size_t MaxSamples>
RingBuffer<MaxSamples>::RingBuffer()
{
    mRingBufferLength = MaxSamples;
    mRingBufferPtr = mBufferStorage.data();
    reset();
}

template<std::size_t MaxSamples>
RingBuffer<MaxSamples>::~RingBuffer()
{
    // No deletion needed - static array
}

template<std::size_t MaxSamples>
void RingBuffer<MaxSamples>::reset()
{
    mReadIdx = mWriteIdx = 0;
    std::fill(mBufferStorage.begin(), mBufferStorage.end(), 0.0);
}

template<std::size_t MaxSamples>
void RingBuffer<MaxSamples>::updateReadIndex()
{
    mReadIdx = updateRingBufferIdx(mReadIdx);
}

template<std::size_t MaxSamples>
void RingBuffer<MaxSamples>::updateWriteIndex()
{
    mWriteIdx = updateRingBufferIdx(mWriteIdx);
}

template<std::size_t MaxSamples>
int RingBuffer<MaxSamples>::updateRingBufferIdx(int index)
{
    index++;
    if (index >= mRingBufferLength) index = 0;
    return index;
}

}   // namespace dspLib

#endif /* RingBuffer_hpp */
```
[ -> Step 2: done 19-11-2025]

### Step 3: Refactor DelayLine to Template

**File**: `dspLib/DelayLine.hpp`

```cpp
#ifndef DelayLine_hpp
#define DelayLine_hpp

#include <stdio.h>
#include <string>
#include "RingBuffer.hpp"
#include "LFO.hpp"
#include "waveforms.hpp"

namespace dspLib {

template<std::size_t MaxSamples>
class DelayLine : public RingBuffer<MaxSamples>
{
public:
    DelayLine();
    DelayLine(int length);
    DelayLine(int length, int delaySamples);
    DelayLine(int length, int delaySamples, int Fs, double frequencyLFO, double modDepth);
    DelayLine(int length, int delaySamples, int Fs, double frequencyLFO, double modDepth, enum dspLib::waveform shape);
    ~DelayLine();

    void reset();
    double processAudio(double sampleIn);
    void setIsModulated(bool isModulated = false);
    void setModDepth(double modDepth = 0.0);
    void setDelaySamples(int delaySamples = 0);
    void updateReadIdx();
    
    LFO<MaxSamples> mLFO;  // LFO Object (also templated)

protected:
    int mDelaySamples;
    bool mIsModulated;
    double mModDepth;
};

// Template implementation
// Full set of constructors and helpers are provided below. Note how the templated
// `LFO<MaxSamples> mLFO` member is default-constructed and then initialized
// in-place when modulation parameters are provided. This avoids manual destructor
// calls and relies on RAII.

template<std::size_t MaxSamples>
DelayLine<MaxSamples>::DelayLine()
    : RingBuffer<MaxSamples>(), mLFO(), mDelaySamples(0),
      mIsModulated(false), mModDepth(0.0)
{
    // Default ctor: RingBuffer ctor sets mRingBufferLength to MaxSamples.
}

template<std::size_t MaxSamples>
DelayLine<MaxSamples>::DelayLine(int length)
    : RingBuffer<MaxSamples>(), mLFO(), mDelaySamples(0),
      mIsModulated(false), mModDepth(0.0)
{
    this->mRingBufferLength = (length > static_cast<int>(MaxSamples)) ? static_cast<int>(MaxSamples) : length;
}

template<std::size_t MaxSamples>
DelayLine<MaxSamples>::DelayLine(int length, int delaySamples)
    : RingBuffer<MaxSamples>(), mLFO(), mDelaySamples(0),
      mIsModulated(false), mModDepth(0.0)
{
    this->mRingBufferLength = (length > static_cast<int>(MaxSamples)) ? static_cast<int>(MaxSamples) : length;
    setDelaySamples(delaySamples);
}

// Delegating constructor: uses default sine waveform when shape not provided
template<std::size_t MaxSamples>
DelayLine<MaxSamples>::DelayLine(int length, int delaySamples, int Fs, double frequencyLFO, double modDepth)
    : DelayLine<MaxSamples>(length, delaySamples, Fs, frequencyLFO, modDepth, dspLib::sine)
{
}

template<std::size_t MaxSamples>
DelayLine<MaxSamples>::DelayLine(int length, int delaySamples, int Fs, double frequencyLFO, double modDepth, enum dspLib::waveform shape)
    : RingBuffer<MaxSamples>(), mLFO(), mDelaySamples(0),
      mIsModulated(false), mModDepth(0.0)
{
    // constrain ring length
    this->mRingBufferLength = (length > static_cast<int>(MaxSamples)) ? static_cast<int>(MaxSamples) : length;

    // base delay samples
    setDelaySamples(delaySamples);

    // modulation handling: if frequencyLFO == 0 => no modulation
    if (frequencyLFO == 0.0)
    {
        mIsModulated = false;
        mModDepth = 0.0;
    }
    else
    {
        mIsModulated = true;
        mModDepth = modDepth;

        // Initialize LFO in-place rather than manually calling destructors.
        // Prefer to call an init() method on LFO to avoid copying the full array
        // if performance is a concern. The example below uses assignment from
        // a temporary which is simple and correct for header-only templates.
        mLFO = LFO<MaxSamples>(Fs, frequencyLFO, shape);
    }
}

template<std::size_t MaxSamples>
DelayLine<MaxSamples>::~DelayLine()
{
    // DO NOT call mLFO.~LFO() — member destructors are invoked automatically.
}

template<std::size_t MaxSamples>
void DelayLine<MaxSamples>::reset()
{
    RingBuffer<MaxSamples>::reset();
    mDelaySamples = 0;
}

template<std::size_t MaxSamples>
double DelayLine<MaxSamples>::processAudio(double sampleIn)
{
    // Calculate read index position (accounts for modulation)
    updateReadIdx();

    // Delay Algorithm (read then write)
    double delayOut = this->mRingBufferPtr[this->mReadIdx];
    this->mRingBufferPtr[this->mWriteIdx] = sampleIn;

    // Update write index
    this->updateWriteIndex();

    // If zero delay, output the input as original logic did
    if (mDelaySamples == 0) delayOut = sampleIn;

    return delayOut;
}

template<std::size_t MaxSamples>
void DelayLine<MaxSamples>::setDelaySamples(int delaySamples)
{
    mDelaySamples = (delaySamples > this->mRingBufferLength) ? this->mRingBufferLength : delaySamples;
}

template<std::size_t MaxSamples>
void DelayLine<MaxSamples>::updateReadIdx()
{
    mReadIdx = mWriteIdx - mDelaySamples;

    if (mIsModulated)
    {
        // modulation changes read index (rounded)
        mReadIdx = static_cast<int>(round(mModDepth * mLFO.out() + mReadIdx));

        // increment LFO read index
        mLFO.updateReadIndex();
    }

    // wrap around
    if (mReadIdx < 0) mReadIdx += this->mRingBufferLength;
}

template<std::size_t MaxSamples>
void DelayLine<MaxSamples>::setIsModulated(bool isModulated)
{
    mIsModulated = isModulated;
}

template<std::size_t MaxSamples>
void DelayLine<MaxSamples>::setModDepth(double modDepth)
{
    mModDepth = modDepth;
}

}   // namespace dspLib

#endif /* DelayLine_hpp */
```
[ -> Step 3: done 22-11-2025]

### Step 4: Refactor LFO to Template

**File**: `dspLib/LFO.hpp`

```cpp
#ifndef LFO_hpp
#define LFO_hpp

#define _USE_MATH_DEFINES
#include <cmath>
#include "waveforms.hpp"
#include <string>
#include "RingBuffer.hpp"

namespace dspLib {

template<std::size_t MaxSamples>
class LFO : public RingBuffer<MaxSamples>
{
public:
    LFO();
    LFO(int Fs, double frequencyLFO);
    LFO(int Fs, double frequencyLFO, enum dspLib::waveform shape);
    ~LFO();
    
    void reset();
    double out();

private:
    enum dspLib::waveform mShape;
    int mIndex;                     // Don't need mIndex>
    void initWaveform();
};

// Template implementation
template<std::size_t MaxSamples>
LFO<MaxSamples>::LFO() 
    : RingBuffer<MaxSamples>(), mShape(SINE), mIndex(0)
{
}

template<std::size_t MaxSamples>
LFO<MaxSamples>::LFO(int Fs, double frequencyLFO)
    : RingBuffer<MaxSamples>(), mShape(SINE), mIndex(0)
{
    int period = static_cast<int>(Fs / frequencyLFO);
    this->mRingBufferLength = (period > MaxSamples) ? MaxSamples : period;
    initWaveform();
}

template<std::size_t MaxSamples>
LFO<MaxSamples>::LFO(int Fs, double frequencyLFO, enum dspLib::waveform shape)
    : RingBuffer<MaxSamples>(), mShape(shape), mIndex(0)
{
    int period = static_cast<int>(Fs / frequencyLFO);
    this->mRingBufferLength = (period > MaxSamples) ? MaxSamples : period;
    initWaveform();
}

template<std::size_t MaxSamples>
LFO<MaxSamples>::~LFO()
{
}

template<std::size_t MaxSamples>
void LFO<MaxSamples>::reset()
{
    RingBuffer<MaxSamples>::reset();
    mIndex = 0;
}

template<std::size_t MaxSamples>
double LFO<MaxSamples>::out()
{
    // MD: Why this? Keep current implementation
    double value = this->mRingBufferPtr[mIndex];
    mIndex++;
    if (mIndex >= this->mRingBufferLength) mIndex = 0;
    return value;
}

template<std::size_t MaxSamples>
void LFO<MaxSamples>::initWaveform()
{
    // Fill buffer with waveform
    for (int i = 0; i < this->mRingBufferLength; i++)
    {
        double phase = 2.0 * M_PI * i / this->mRingBufferLength;
        this->mRingBufferPtr[i] = getWaveform(phase, mShape);
    }
}

}   // namespace dspLib

#endif /* LFO_hpp */
```
[ -> Step 4: done 17-11-2025]


### Step 5: Refactor AllPass to Template

**File**: `dspLib/AllPass.hpp`

```cpp
#ifndef AllPass_hpp
#define AllPass_hpp

#include "DelayLine.hpp"

namespace dspLib {

template<std::size_t MaxSamples>
class AllPass : public DelayLine<MaxSamples>
{
public:
    AllPass();
    AllPass(int length);
    AllPass(int length, int delaySamples);
    AllPass(int length, int delaySamples, int Fs, double modFreq, double modDepth);
    ~AllPass();

    void setFeedbackCoefficient(double coeff);
    double processAudio(double sampleIn);
    void reset();

private:
    double mFeedbackCoeff;
};

// Template implementation
template<std::size_t MaxSamples>
AllPass<MaxSamples>::AllPass()
    : DelayLine<MaxSamples>(), mFeedbackCoeff(0.5)
{
}

template<std::size_t MaxSamples>
AllPass<MaxSamples>::AllPass(int length)
    : DelayLine<MaxSamples>(length), mFeedbackCoeff(0.5)
{
}

template<std::size_t MaxSamples>
AllPass<MaxSamples>::AllPass(int length, int delaySamples)
    : DelayLine<MaxSamples>(length, delaySamples), mFeedbackCoeff(0.5)
{
}

template<std::size_t MaxSamples>
AllPass<MaxSamples>::AllPass(int length, int delaySamples, int Fs, double modFreq, double modDepth)
    : DelayLine<MaxSamples>(length, delaySamples, Fs, modFreq, modDepth), mFeedbackCoeff(0.5)
{
}

template<std::size_t MaxSamples>
AllPass<MaxSamples>::~AllPass()
{
}

template<std::size_t MaxSamples>
void AllPass<MaxSamples>::setFeedbackCoefficient(double coeff)
{
    mFeedbackCoeff = coeff;
}

template<std::size_t MaxSamples>
double AllPass<MaxSamples>::processAudio(double sampleIn)
{
    // AllPass: y[n] = -x[n] + a*x[n-d] + a*y[n-d]
    double delayedSample = this->mRingBufferPtr[this->mReadIdx];
    double output = -sampleIn + mFeedbackCoeff * delayedSample;
    
    // Write: input + feedback * output
    this->mRingBufferPtr[this->mWriteIdx] = sampleIn + mFeedbackCoeff * output;
    
    this->updateReadIdx();
    this->updateWriteIndex();
    
    return output;
}

template<std::size_t MaxSamples>
void AllPass<MaxSamples>::reset()
{
    DelayLine<MaxSamples>::reset();
    mFeedbackCoeff = 0.5;
}

}   // namespace dspLib

#endif /* AllPass_hpp */
```
[ -> Step 5: done 22-11-2025]

### Step 6: Update ReverbZ to Use Templates
[Skipped - Implemented step 7 directly.]

**File**: `OwnProjects/_projLib/ReverbZ.hpp`

Add the config header and change template parameters:

```cpp
#pragma once
#ifndef ReverbZ_hpp
#define ReverbZ_hpp

// Include project config for buffer size
#include "../ReverbZpatch/ReverbZpatch_config.hpp"

// Include used dspLib components (now templated)
#include "../../dspLib/AllPass.hpp"
#include "../../dspLib/DelayLine.hpp"  
#include "../../dspLib/DeZipper.hpp"           
#include "../../dspLib/OnePoleFilter.hpp"
#include "../../dspLib/Saturator.hpp"
#include "../../dspLib/mathUtils.hpp"

using namespace dspLib;

namespace projLib {

class ReverbZ {
    public:
        ReverbZ(int sampleRate);
        ~ReverbZ();

        // ... (rest of public interface unchanged)

    private:
        // Use templated classes with project-defined buffer size
        DelayLine<DSPLIB_MAX_BUFFER_SIZE> mPredelay;
        AllPass<DSPLIB_MAX_BUFFER_SIZE> mInputAllpass1;
        AllPass<DSPLIB_MAX_BUFFER_SIZE> mInputAllpass2;
        AllPass<DSPLIB_MAX_BUFFER_SIZE> mInputAllpass3;
        AllPass<DSPLIB_MAX_BUFFER_SIZE> mInputAllpass4;
        AllPass<DSPLIB_MAX_BUFFER_SIZE> mModAllpass1;
        AllPass<DSPLIB_MAX_BUFFER_SIZE> mModAllpass2;
        
        DelayLine<DSPLIB_MAX_BUFFER_SIZE> mTankDelay1;
        DelayLine<DSPLIB_MAX_BUFFER_SIZE> mTankDelay2;
        DelayLine<DSPLIB_MAX_BUFFER_SIZE> mTankDelay3;
        DelayLine<DSPLIB_MAX_BUFFER_SIZE> mTankDelay4;
        
        AllPass<DSPLIB_MAX_BUFFER_SIZE> mTankAllpass5;
        AllPass<DSPLIB_MAX_BUFFER_SIZE> mTankAllpass6;
        AllPass<DSPLIB_MAX_BUFFER_SIZE> mTankAllpass7;
        AllPass<DSPLIB_MAX_BUFFER_SIZE> mTankAllpass8;
        AllPass<DSPLIB_MAX_BUFFER_SIZE> mTankAllpass9;
        AllPass<DSPLIB_MAX_BUFFER_SIZE> mTankAllpass10;
        
        // ... (rest of private members unchanged)
};

}   // namespace projLib

#endif /* ReverbZ_hpp */
```
[Skipped - Implemented step 7 directly.]

### Step 7: Make `ReverbZ` Templated (recommended for reuse)

If you want `ReverbZ` to be reusable across multiple projects that choose different buffer sizes, make the `ReverbZ` class itself a template parameterized by the buffer size. This propagates the compile-time `MaxSamples` through all internal dsp objects and keeps the API portable.

Recommended pattern: header + `.tpp` (implementation) included at end of header.

1) Change `ReverbZ.hpp` declaration to a template:

```cpp
// ReverbZ.hpp
#pragma once
#include "../../dspLib/AllPass.hpp"
#include "../../dspLib/DelayLine.hpp"
#include "../../dspLib/DeZipper.hpp"
#include "../../dspLib/OnePoleFilter.hpp"
#include "../../dspLib/Saturator.hpp"
#include "../../dspLib/mathUtils.hpp"

namespace projLib {

template<std::size_t MaxSamples>
class ReverbZ
{
public:
    ReverbZ(int sampleRate);
    ~ReverbZ();

    void init();
    void processAudioMono(double inputSample);
    void processAudioStereo(double inputSampleL, double inputSampleR);
    void setControlParameters(...);

    double mOutL, mOutR, mOutMono;

private:
    void processAudioPrivate(double inputSample);

    // templated dsp members
    dspLib::DelayLine<MaxSamples>   mPredelay;
    dspLib::AllPass<MaxSamples>     mInputAllpass1;
    dspLib::AllPass<MaxSamples>     mInputAllpass2;
    // ... rest of members use <MaxSamples>
};

} // namespace projLib

#include "ReverbZ.tpp" // implementation (template definitions)
```

2) Move `ReverbZ.cpp` implementation into `ReverbZ.tpp` and adapt definitions:

```cpp
// ReverbZ.tpp (implementation visible to all TUs)
template<std::size_t MaxSamples>
projLib::ReverbZ<MaxSamples>::ReverbZ(int sampleRate)
 : mPredelay(), mInputAllpass1(), /* ... */
{
    mFs = sampleRate;
    init();
}

template<std::size_t MaxSamples>
projLib::ReverbZ<MaxSamples>::~ReverbZ() {}

// ... other member definitions
```

Keep `ReverbZ.tpp` included at the end of the header so the template definitions are visible to every translation unit that instantiates `ReverbZ<...>`.

3) Project usage (example - `ReverbZpatch`)

Create a small project config header that defines the size and include it before using the template type:

```cpp
// OwnProjects/ReverbZpatch/dspconfig.hpp
#pragma once
#include <cstddef>
constexpr std::size_t DSPLIB_MAX_BUFFER_SIZE = 48000;
```

Then in your application source:

```cpp
#include "dspconfig.hpp"
#include "../_projLib/ReverbZ.hpp"

using ReverbZ_t = projLib::ReverbZ<DSPLIB_MAX_BUFFER_SIZE>;

// construct after system/hardware init (avoid static global construction)
ReverbZ_t* reverbz = nullptr;

int main() {
    // init hardware
    reverbz = new ReverbZ_t(FS_REVERBZ);
}
```

4) Explicit instantiation (optional) [NOT DONE AT THE MOMENT, MD 23-11-2025]

If you want to avoid the template being instantiated in many TUs (to reduce code bloat), you can explicitly instantiate a specific size in one translation unit:

```cpp
// ReverbZ_explicit_inst.cpp
#include "dspconfig.hpp"
#include "../_projLib/ReverbZ.hpp"

template class projLib::ReverbZ<DSPLIB_MAX_BUFFER_SIZE>;
```

And in headers (or other TUs) add `extern template class projLib::ReverbZ<DSPLIB_MAX_BUFFER_SIZE>;` to avoid implicit instantiation.

5) Notes & tradeoffs

- Header-only (`.tpp`) approach: flexible, reusable for any size, simplest for multi-project reuse.
- Explicit instantiation: one compiled copy per size, reduces duplicates but requires you to manage instantiations for each size used.
- Memory: templated dsp objects allocate `MaxSamples * sizeof(double)` per buffer. Carefully choose `MaxSamples` (or switch to `float`) to fit target RAM.

6) Update the Testing Checklist

Add to the checklist:
- [ ] Compile `ReverbZ` template(s) for the project sizes you need (or add explicit instantiation TU)
- [ ] Verify that `ReverbZ` can be instantiated with different `DSPLIB_MAX_BUFFER_SIZE` values in other projects


---

## Key Benefits

1. **No dynamic allocation**: All buffers allocated at compile-time on the stack/static storage
2. **Predictable memory**: No fragmentation or allocation failures at runtime
3. **Zero overhead**: Template specialization produces optimal code per buffer size
4. **Embedded-friendly**: No `new`/`delete`, suitable for hard real-time systems
5. **Type-safe**: Buffer size is part of the type system

---

## Important Notes

### Memory Considerations
- Each object now reserves `MaxSamples * sizeof(double)` bytes statically
- With `MaxSamples = 48000` and `sizeof(double) = 8`, each buffer = ~384 KB
- Stack size may need adjustment if creating many objects

### Constructor Changes
- Runtime buffer length selection is replaced with compile-time template parameter
- Remove calls like `RingBuffer(48000)` → use `RingBuffer<48000>()`
- Non-templated constructors become default constructors or can accept length hints (checked at compile-time)

### Migration Path (if gradual needed)
1. Keep old dynamic versions in separate files (e.g., `RingBuffer_dynamic.hpp`)
2. Create template versions alongside
3. Gradually migrate projects to templates
4. Remove dynamic versions once all projects migrated

### Adjusting Buffer Size per Project
Different projects can use different buffer sizes:
- **ReverbZpatch**: `const int DSPLIB_MAX_BUFFER_SIZE = 48000;`
- **Another project**: `const int DSPLIB_MAX_BUFFER_SIZE = 32000;`

Just include the project-specific config header.

---

## Testing Checklist

- [ ] Compile dspLib with new template RingBuffer
- [ ] Compile ReverbZ with templated DelayLine/AllPass
- [ ] Compile ReverbZpatch (uses ReverbZ)
- [ ] Verify no `new`/`delete` calls in object files (use `nm` or `strings`)
- [ ] Test audio processing (same functionality as before)
- [ ] Measure memory usage (should be constant)
- [ ] Run existing tests or unit tests if available

---

## Backward Compatibility Note

If other projects still use the old dynamic allocation API, you have two options:

1. **Create a compatibility wrapper**: A non-template class that wraps the template with default size
2. **Maintain both versions**: Keep dynamic and template versions available

For this refactor, it's recommended to update all projects to use templates for consistency.
