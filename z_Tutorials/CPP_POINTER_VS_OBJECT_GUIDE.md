# C++ Object vs Pointer: Dot (`.`) vs Arrow (`->`) Notation Guide

This guide explains the difference between working with objects directly versus through pointers in C++, and when to use dot (`.`) versus arrow (`->`) notation for member access.

## Quick Reference

| Declaration | Storage | Member Access | Example |
|------------|---------|---------------|---------|
| `ReverbZ_t obj(args);` | Stack (automatic) | Dot (`.`) | `obj.init();` |
| `ReverbZ_t* ptr = new ReverbZ_t(args);` | Heap (manual) | Arrow (`->`) | `ptr->init();` |
| `ReverbZ_t& ref = obj;` | Reference to existing | Dot (`.`) | `ref.init();` |

## Dot (`.`) Notation — Direct Object Access

Use the **dot** when you have the actual object (not a pointer).

### Stack-allocated object
```cpp
// Declaration: object on the stack
ReverbZ_t reverbz(FS_REVERBZ);

// Member access: use dot
reverbz.init();
reverbz.processAudioStereo(in[0][i], in[1][i]);
float outL = reverbz.mOutL;
float outR = reverbz.mOutR;
```

**Characteristics:**
- Object is created when the variable is declared
- Automatically destroyed when it goes out of scope (e.g., end of function or program)
- No manual memory management needed (`new`/`delete` not required)
- Simpler and safer for most use cases

### Reference (acts like an object)
```cpp
ReverbZ_t reverbz(FS_REVERBZ);
ReverbZ_t& ref = reverbz;  // reference to the same object

// Member access: still use dot
ref.init();
ref.processAudioStereo(in[0][i], in[1][i]);
```

## Arrow (`->`) Notation — Pointer Access

Use the **arrow** when you have a pointer to an object.

### Heap-allocated object with `new`
```cpp
// Declaration: pointer to object on the heap
ReverbZ_t* reverbz = new ReverbZ_t(FS_REVERBZ);

// Member access: use arrow
reverbz->init();
reverbz->processAudioStereo(in[0][i], in[1][i]);
float outL = reverbz->mOutL;
float outR = reverbz->mOutR;

// Manual cleanup required
delete reverbz;  // must free memory when done
```

**Characteristics:**
- Object is allocated on the heap with `new`
- Pointer variable stores the memory address of the object
- You must manually `delete` the object to free memory
- Object persists until explicitly deleted (not tied to scope)

### What `->` really means
The arrow operator `->` is syntactic sugar for dereferencing and then accessing:
```cpp
ptr->member;     // shorthand
(*ptr).member;   // equivalent explicit form
```

## Common Mistakes and Fixes

### Mistake 1: Most Vexing Parse (function declaration instead of object)
```cpp
// WRONG: This declares a function, not an object!
ReverbZ_t* reverbz(FS_REVERBZ);
// Compiler sees: function named "reverbz" that returns ReverbZ_t* and takes int

// FIX Option 1: Use assignment
ReverbZ_t* reverbz = new ReverbZ_t(FS_REVERBZ);

// FIX Option 2: Use uniform initialization
ReverbZ_t* reverbz{new ReverbZ_t(FS_REVERBZ)};

// FIX Option 3: Stack allocation (no pointer)
ReverbZ_t reverbz(FS_REVERBZ);
```

### Mistake 2: Using arrow on an object (not a pointer)
```cpp
ReverbZ_t reverbz(FS_REVERBZ);  // object, not pointer
reverbz->init();  // WRONG: cannot use -> on an object

// FIX: use dot
reverbz.init();
```

### Mistake 3: Using dot on a pointer
```cpp
ReverbZ_t* reverbz = new ReverbZ_t(FS_REVERBZ);
reverbz.init();  // WRONG: cannot use . on a pointer

// FIX: use arrow
reverbz->init();
```

### Mistake 4: Null pointer dereference
```cpp
ReverbZ_t* reverbz = nullptr;  // pointer not pointing to anything
reverbz->init();  // CRASH: dereferencing null pointer

// FIX: allocate before use
reverbz = new ReverbZ_t(FS_REVERBZ);
reverbz->init();  // safe now

// Or use a guard in audio callback:
if (reverbz) {
    reverbz->processAudioStereo(in[0][i], in[1][i]);
}
```

## When to Use Each Approach

### Use stack allocation (object with dot) when:
- Object lifetime matches scope (e.g., lives for entire program, or only in a function)
- You want automatic cleanup (no manual `delete` needed)
- Object size is reasonable (not multiple megabytes on the stack)
- Simpler code is preferred

**Example (recommended for ReverbZ):**
```cpp
// Global or in main:
ReverbZ_t reverbz(FS_REVERBZ);

int main() {
    patch.Init();
    reverbz.init();  // allocate SDRAM buffers
    patch.StartAudio(AudioCallback);
    while(1) {}
}

void AudioCallback(...) {
    reverbz.processAudioStereo(in[0][i], in[1][i]);
    out[0][i] = reverbz.mOutL;
    out[1][i] = reverbz.mOutR;
}
```

### Use heap allocation (pointer with arrow) when:
- Object lifetime needs to be controlled independently of scope
- Object may be created/destroyed dynamically at runtime
- You need polymorphism (base class pointer to derived object)
- Object is very large and stack size is limited

**Example:**
```cpp
// Global pointer
ReverbZ_t* reverbz = nullptr;

int main() {
    patch.Init();
    reverbz = new ReverbZ_t(FS_REVERBZ);  // create on heap
    reverbz->init();
    patch.StartAudio(AudioCallback);
    while(1) {}
    // delete reverbz;  // cleanup (if program ever exits)
}

void AudioCallback(...) {
    if (reverbz) {  // safety check
        reverbz->processAudioStereo(in[0][i], in[1][i]);
        out[0][i] = reverbz->mOutL;
        out[1][i] = reverbz->mOutR;
    }
}
```

## Summary Table

| Aspect | Object (`.`) | Pointer (`->`) |
|--------|-------------|----------------|
| Declaration | `Type obj(args);` | `Type* ptr = new Type(args);` |
| Member access | `obj.member` | `ptr->member` |
| Storage | Stack (automatic) | Heap (manual) |
| Lifetime | Scope-based | Until `delete` called |
| Cleanup | Automatic | Manual (`delete ptr;`) |
| Safety | Safer (no null/dangling) | Requires null checks |
| Performance | Slightly faster (no indirection) | Indirection overhead |

## Practical Recommendation for ReverbZ

For your `ReverbZ` use case, **stack allocation with dot notation** is recommended:

```cpp
// In ReverbZpatch.cpp (global or in main)
ReverbZ_t reverbz(FS_REVERBZ);

int main() {
    patch.Init();
    reverbz.init();  // dot notation
    patch.StartAudio(AudioCallback);
    while(1) {}
}

void AudioCallback(...) {
    reverbz.processAudioStereo(in[0][i], in[1][i]);  // dot notation
    out[0][i] = reverbz.mOutL;
    out[1][i] = reverbz.mOutR;
}
```

This is simpler, safer, and avoids the need for null checks in the audio callback.
