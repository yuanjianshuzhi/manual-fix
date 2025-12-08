# 修复反编译 C++ 代码编译错误的指导文档

## Introduction

This document provides guidance on fixing compilation errors in decompiled C++ code, based on common patterns and examples from the FixAgent prompt library. The examples are derived from real-world fixes for MSVC (cl) compilation issues in decompiled binaries.

## 常见修复模式 (Common Fix Patterns)

### a. Undeclared Identifier (C2065)

**Error:** 'qword_141A25030': undeclared identifier

**Strategy:** Replace direct references to undeclared variables with dereferenced pointers if they represent memory locations.

**Example:**

```cpp
// Before
sub_1404443C0_ptr((_QWORD)(&qword_141A25030), ...);

// After
sub_1404443C0_ptr((_QWORD)(&(*unk_141A25030)), ...);
```



### b. SIMD Intrinsic Instructions

**Error:** Issues with SSE/AVX instructions in decompiled code.

**Strategy:** Replace complex SIMD operations with standard memory copy operations to simplify and ensure compatibility.

**Example:**

```cpp
// Before: Complex SIMD loads and stores
si128 = _mm_load_si128((const __m128i *)&tm_1.tm_mon);
*(__m128i *)&tm->tm_sec = _mm_load_si128((const __m128i *)&tm_1);

// After: Simple struct assignment
*tm = tm_1;
```

### c. Invalid Conversion (Integer to Pointer)

**Error:** Cannot convert integer to pointer type.

**Strategy:** Modify function typedefs to use generic types like `uint64_t` or `void*` instead of specific pointer types.

**Example:**

```cpp
// Before: typedef void (*func_t)(double* a1);
func_ptr(1); // Error

// After: typedef void (*func_t)(uint64_t a1);
func_ptr(1);
```

### d. Subscript Requires Array or Pointer Type (C2109)

**Error:** Attempting to use subscript on a non-array type.

**Strategy:** Remove unnecessary dereference operators and cast to appropriate pointer types.

**Example 1:**

```cpp
// Before
v1 = (*qword_142FF8270)[index];

// After
v1 = ((uint64_t*)qword_142FF8270)[index];
```

**Example 2:**

```cpp
// Before
uint64 v5;
v5[0] = 1;

// After
uint64 v5;
((uint64_t*)v5)[0] = 1;
```

### e. Complex Pointer Arithmetic Syntax Errors (C2059/C2100)

**Error:** Syntax errors in deeply nested pointer expressions.

**Strategy:** Simplify expressions by removing unnecessary dereferences and using direct pointer arithmetic.

**Example:**

```cpp
// Before
v1 = *(__int64*)((char*)&(*qword_ptr)[16 * v5 - 5] + v140);

// After
v1 = *(__int64*)((char*)qword_ptr + (16 * v5 - 5) + v140);
```

### f. STACK Undeclared (IDA Pseudocode Residue)

**Error:** References to STACK[] which are IDA artifacts.

**Strategy:** Remove or comment out STACK references as they are not valid C++ code.

**Example:**

```cpp
// Before
STACK[0x28] = v5;
v6 = STACK[0x30];

// After
// Removed IDA STACK references
// STACK[0x28] = v5;
// v6 = STACK[0x30];
```

### e. 一些结构的替换

__int128 可以替换成  int128

### g. 对于变量转换错误的问题 ERROR C2664

手动修复的话，只需要将函数定义里面的参数类型，强制付给调用是的参数就行。

例如 v239[1] = "Upd ii to VLAM:"; 报错

修改为：其中v239是uint64数组

v239[1] = (uint64)"Upd ii to VLAM:";


### h. 函数参数数量不对

如果是参数少了，就在后面增加,0,0,0直到满足位置

如果是参数多了，就在函数定义的参数后面添加,...

例如：typedef unsigned __int64 (*sub_1411C5E90_t)(const char *a1, __int64 a2,...);

### i.关于 __m128, __m128i, __m128d的转换问题

他们三个之间的转换需要使用函数，不能靠指定类型完成转换

源类型 ↓ / 目标类型 →,__m128 (单精度浮点向量),__m128d (双精度浮点向量),__m128i (整数向量)

 __m128,            N/A,                   _mm_castps_pd,          _mm_castps_si128
 
 __m128d,           _mm_castpd_ps,         N/A,                    _mm_castpd_si128
 
__m128i,            _mm_castsi128_ps,      _mm_castsi128_pd,       N/A

同时可以使用_mm_cvtsd_si64将__m128d转换成long long，这个通常出现在需要&运算的时候

如果要将__m128d赋值给其他变量，需要如下修改：

例如：

*(int128 *)((char *)(*xmmword_142D3E5F8_ptr) + 8 * v99) = (int128)_mm_add_pd...

修改为：

\*(__m128d\*)((char *)(*xmmword_142D3E5F8_ptr) + 8 * v99) = _mm_add_pd...


如果需要对一个__m128/__m128i/__m128d的变量赋值0，应该修改为：

错误： v67 = 0；

修改为：v67 = {0,0}；


## 关键规则 (Critical Rules)

1. **Short & Unique SEARCH:** Use only 2-4 lines of unique code for SEARCH blocks in replacements. Avoid large chunks of surrounding code.

2. **Stop Type Fighting:** When encountering conflicts between `int64*`, `uint64*`, and `_QWORD*`, redefine variables or function arguments as `void*` or `unsigned __int64` instead of adding more casts.

3. **Handle Missing Definitions:**
   - For undefined `_TEB`: Cast `NtCurrentTeb()` to `(char*)` and add byte offsets manually.
   - For LNK2005 "symbol already defined": Delete duplicate function implementations.

4. **Return Values:** Ensure non-void functions end with appropriate return statements (e.g., `return 0;`).

## Additional Guidelines

- **Preserve Semantics:** Minimize changes and preserve original logic as much as possible.
- **Focus on Errors:** Only fix compilation errors; ignore warnings.
- **Include Management:** Do not arbitrarily add or remove `#include` directives.
- **Output Format:** Precede each fix with the target file path, and ensure SEARCH/REPLACE blocks are precise.

This guide is based on patterns observed in decompiled code fixes. Always test changes with MSVC compilation to verify correctness.
