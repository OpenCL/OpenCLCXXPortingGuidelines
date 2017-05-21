# <a name="title"></a>OpenCL C to OpenCL C++ Porting Guidelines

May 16, 2017

Editors:

* [Jakub Szuppe, Stream HPC](https://streamhpc.com)

This document is a set of guidelines for developers who know OpenCL C and plan to
port their kernels to OpenCL C++, and therefore they need to know the main
differences between those two kernel languages.
The focus is not on highlighting all the differences, but rather on exposing
and explaining those that are the most important, and those that may cause
hard-to-detect bugs when porting from OpenCL C to OpenCL C++.

Comments, suggestions for improvements, and contributions are most welcome.

**[Differences](#S-Differences)**:

* [OpenCL C++ Programming Language](#S-OpenCLCXX):
  * [OpenCL C Vector Literals](#S-OpenCLCXX-VectorLiterals)
  * [<code>bool<i>N</i></code> Type](#S-OpenCLCXX-BoolNType)
  * [End Of Explicit Named Address Spaces](#S-OpenCLCXX-EndOfExplicitNamedAddressSpaces)
  * [Kernel Function Restrictions](#S-OpenCLCXX-KernelRestrictions)
  * [Kernel Parameter Restrictions](#S-OpenCLCXX-KernelParamsRestrictions)
  * [General Restrictions](#S-OpenCLCXX-GeneralRestrictions)
* [OpenCL C++ Standard Library](#S-OpenCLCXXSTL):
  * [Namespace cl::](#S-OpenCLCXXSTL-NamespaceCL)
  * [Conversions Library (`convert_*()`)](#S-OpenCLCXXSTL-ConversionsLibrary)
  * [Reinterpreting Data Library (<code>as&#95;<i>type</i>()</code>)](#S-OpenCLCXXSTL-ReinterpretingDataLibrary)
  * [Address Spaces Library](#S-OpenCLCXXSTL-AddressSpacesLibrary)
  * [Marker Types](#S-OpenCLCXXSTL-MarkerTypes)
  * [Images and Samplers Library](#S-OpenCLCXXSTL-ImagesAndSamplersLibrary)
  * [Pipes Library](#S-OpenCLCXXSTL-PipesLibrary)
  * [Device Enqueue Library](#S-OpenCLCXXSTL-DeviceEnqueueLibrary)
  * [Relational Functions](#S-OpenCLCXXSTL-RelationalFunctions)
  * [Vector Data Load and Store Functions](#S-OpenCLCXXSTL-VectorDataLoadandStoreFunctions)
  * [Atomic Operations Library](#S-OpenCLCXXSTL-AtomicOperationsLibrary)
* [OpenCL C++ Compilation Process](#S-OpenCLCXXCompilation):
  * [OpenCL C++ Compilation to SPIR-V](#S-OpenCLCXXCompilationToSPIRV)
  * [Building program created from SPIR-V](#S-OpenCLCXXCompilationBuildSPIRV)

**[Bibliography](#S-Bibliography)**

# <a name="S-Differences"></a>Differences

## <a name="S-OpenCLCXX"></a>OpenCL C++ Programming Language

### <a name="S-OpenCLCXX-VectorLiterals"></a>OpenCL C Vector Literals

Vector literals, expression used for creating vectors from a list of scalars,
vectors or a mixture thereof, known from OpenCL C are not part of the OpenCL C++
kernel language.

In OpenCL C++ vector types can be initialized like any other class - using
constructors. For example, the following are available for `float4`:

```cpp
float4(float, float, float, float)
float4(float2, float, float)
float4(float, float2, float)
float4(float, float, float2)
float4(float2, float2)
float4(float3, float)
float4(float, float3)
float4(float)
```

##### Note
> In OpenCL C++ vector literals are NOT evaluated as user might expect,
unfortunately, they never cause compilation errors.

Vector literals in OpenCL C++ are not evaluated as user might expect.
In OpenCL C++ expression `(int4)(1, 2, 3, 4)` is evaluated to `(int4)4`.
This happens because of how comma operator works: every value enclosed in
parentheses except for the last is discarded, and then scalar-to-vector
conversion is used for `4`.

In certain situations vector literals in OpenCL C++ code can cause warnings
during compilation, but they do not cause compilation errors.

#### Solution

Do not use vector literals. Replace them with vector constructors.

#### Examples, bad

```cpp
int4 i = (int4)(1, 2, 3, 4);
// This expression will be evaluated to (int4)4,
// and i will be (4, 4, 4, 4).
// In OpenCL C++ compiler (clang) provided by Khronos
// it causes 'expression result unused' warnings.

int4 i = (int4)(cl::max(0, 1), cl::max(0, 2), cl::max(0, 3), cl::max(0, 4))
// This expression will be evaluated to (int4)4,
// and i will be (4, 4, 4, 4).
// In OpenCL C++ compiler (clang) provided by Khronos
// it DOES NOT cause any warnings.
```

#### Examples, correct

```cpp
uint4 u = uint4(1); //  u will be (1, 1, 1, 1)
int4  i = int4{-1, -2, 3, 4} // i will be (-1, -2, 3, 4)

// in each case f will be (1.0f, 2.0f, 3.0f, 4.0f)
float4 f = float4(1.0f, 2.0f, 3.0f, 4.0f);
float4 f = float4(float2(1.0f, 2.0f), float2(3.0f, 4.0f));
float4 f = float4(1.0f, float2(2.0f, 3.0f), 4.0f);
```

### <a name="S-OpenCLCXX-BoolNType"></a><code>bool<i>N</i></code> Type

OpenCL C++ introduces new built-in vector type: `boolN` (where `N` is 2, 3, 4, 8, or 16). This addition change
resolves problem with using the relational (`<`, `>`, `<=`, `>=`, `==`, `!=`) and the logical operators
(`!`, `&&`, `||`) with built-in vector types.

In OpenCL C for built-in vector types the relational and the logical operators return a vector signed
integer type of the same size as the source operands. In OpenCL C++ it was simpliefied and
those operators return `boolN` for vector types and `bool` for scalars.

[The OpenCL C 2.0 Specification](#https://www.khronos.org/registry/OpenCL/specs/opencl-2.0-openclc.pdf#page=27)
on the results of the relational operators:
>The result is a scalar signed integer of type `int` if the source operands are scalar and a vector
signed integer type of the same size as the source operands if the source operands are vector
types.  Vector source operands of type `charn` and `ucharn` return a `charn` result; vector
source operands of type `shortn` and `ushortn` return a `shortn` result; vector source
operands of type `intn`, `uintn` and `floatn` return an `intn` result; vector source operands
of type `longn`, `ulongn` and `doublen` return a `longn` result.

>For scalar types, the relational operators shall return `0` if the specified relation is `false` and `1` if
the specified relation is `true`. For vector types, the relational operators shall return `0` if the specified
relation is `false` and `–1` (i.e. all bits set) if the specified relation is `true`. The relational
operators always return `0` if either argument is not a number (`NaN`).


Including `boolN` vector types in OpenCL C++ also caused changes in signatures and/or behavior of
built-in relational functions like: `all()`, `any()` and `select()`.
See [Relational Functions](#S-OpenCLCXXSTL-RelationalFunctions) section for more details.

#### Examples

```cpp
bool2 b = bool2(1 == 0); // { false, false }

// In OpenCL C: int b = 2 > 1, and b is 1
bool b = 2 > 1 // true

// In OpenCL C: int b = 2 > 1, and b is 0
bool b = 2 == 1 // false

// OpenCL C-related note:
// -1 for signed integer type means that all bits are set

// In OpenCL C: int2 b = (uint2)(0, 1) > (uint2)(0, 0),
// and b is { 0, -1 }
bool2 b = uint2(0, 1) > uint2(0, 0); // { false, true }

// In OpenCL C: long2 b = (ulong2)(0, 0) > (ulong2)(0, 0),
// and b is { 0, 0 }
bool2 b = ulong2(0, 0) > ulong2(0, 0); // { false, false }

// In OpenCL C: long2 b = (long2)(1, 1) > (long2)(0, 0),
// and b is { -1, -1 }
bool2 b = long2(1, 1) > long2(0, 0); // { true, true }
```

```cpp
#include  <opencl_relational>

// In OpenCL C: int2 b = isnan((float2)(0.0f)),
// and b is { 0, 0 }
bool2 b = isnan(float2(0.0f)) // { false, false }

// In OpenCL C: long2 b = isfinite((double2)(0.0))
// and b is { -1, -1 }
bool2 b = isfinite(double2(0.0)) // { true, true }
```

#### OpenCL C++ Specification References

* [OpenCL C++ Programming Language: Expressions](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#expressions)

### <a name="S-OpenCLCXX-EndOfExplicitNamedAddressSpaces"></a>End Of Explicit Named Address Spaces

[OpenCL C++ 1.0 Specification in Address Spaces section](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#address-spaces)
says:
>The OpenCL C++ kernel language doesn’t introduce any explicit named address spaces, but they are
implemented as part of the standard library described in Address Spaces Library section.
There are 4 types of memory supported by all OpenCL devices: global, local, private and constant.
The developers should be aware of them and know their limitations.

That means that instead of using keywords `global`, `constant`, `local`, and `private`, in order
to explicitly specify address space for variable or pointer you have to use address space pointers
and address space storage classes.

##### Note
> Go to [Address Spaces Library](#S-OpenCLCXXSTL-AddressSpacesLibrary) section of
The Porting Guidelines to read more about address space pointers and address space storage classes.

It is still possible for OpenCL C++ compiler to deduce an address space based on the scope where
an object is declared:

* If a variable is declared in program scope, with `static` or `extern` specifier and the standard
library storage class (see
[Explicit address space storage classes](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#explicit-address-space-storage-classes)
section) is not used, the variable is allocated in the global memory of a device.
* If a variable is declared in function scope, without static specifier and the standard library storage class
(see [Explicit address space storage classes](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#explicit-address-space-storage-classes)
section) is not used, the variable is allocated in the private memory of a device.

#### OpenCL C++ Specification References

* [OpenCL C++ Programming Language: Address Spaces](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#address-spaces)
* [OpenCL C++ Standard Library: Address Spaces Library](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#address-spaces-library)

#### Examples, bad (OpenCL C-style)

```cpp
// Compilation error, "global" address space is not defined
// in OpenCL C++ kernel language
kernel void example_kernel(global int * input)
{
  // Compilation error, "local" address space is not defined
  // in OpenCL C++ kernel language
  local int array[256];
  // ...
}

// Compilation error, "constant" address space is not defined
// in OpenCL C++ kernel language
kernel void example_kernel(constant int * input)
{
  // Compilation error, "private" address space is not defined
  // in OpenCL C++ kernel language
  private int x;
  // ...
}
```

#### Examples, correct (OpenCL C++)

```cpp
#include <opencl_memory>
#include <opencl_work_item>

kernel void example_kernel(cl::global_ptr<int[]> input)
{
  cl::local<int[256]> array;

  uint gid = cl::get_global_id(0);
  array[gid] = input[gid];
  // ...
}

kernel void example_kernel(cl::constant_ptr<int[]> input)
{
  int x = 0;
  // ...
}

int y; // Allocated in global memory
static int z; // Allocated in global memory

kernel void example_kernel(cl::constant_ptr<int[]> input)
{
  int x = 0; // Allocated in private memory
  static cl::global<int> w; // Allocated in global memory
  // ...
}
```

##### Note
> More examples on address spaces can be found in subsections
[3.4.5. Restrictions](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#restrictions-2) and
[3.4.6. Examples](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#examples-3) of section
[Address Spaces Library](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#address-spaces-library) in
[OpenCL C++ specification](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html).

### <a name="S-OpenCLCXX-KernelRestrictions"></a>Kernel Function Restrictions

Since OpenCL C++ kernel language is based on C++14 several restrictions were defined for
kernel function to make it resemble kernel function known from OpenCL C:

* A kernel functions are by implicitly declared as extern "C".
* A kernel function cannot be overloaded.
* A kernel function cannot be template function.
* A kernel function cannot be called by another kernel function.
* A kernel function cannot have parameters specified with default values.
* A kernel function must have the return type void.
* A kernel function cannot be called main.

##### Note
> Compared to OpenCL C in OpenCL C++ you cannot call a kernel function from another kernel function.

#### OpenCL C++ Specification References

* [OpenCL C++ Programming Language: Kernel Functions](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#kernel-functions)

#### Examples, bad

```cpp
// A kernel function cannot be template function.
template<class T>
kernel void example_kernel(cl::global_ptr<T[]> input, uint size)
{ /* ... */ }

// A kernel function cannot have parameters specified with default values.
kernel void foo(cl::global_ptr<uint[]> input, uint size = 10)
{ /* ... */ }

kernel void bar(cl::global_ptr<uint[]> input, uint size)
{
  // A kernel function cannot be called by another kernel function.
  foo(input, size);
}

// A kernel function cannot be overloaded.
kernel void bar(cl::global_ptr<float[]> input, uint size)
{ /* ... */ }
```

#### Examples, correct

```cpp
template<class T>
void function_template(cl::global_ptr<T[]> input, uint size)
{ /* ... */ }

// Specialization for T = float
template<>
void function_template(cl::global_ptr<float[]> input, uint size)
{ /* ... */ }

kernel void kernel_uint(cl::global_ptr<uint[]> input, uint size)
{
  function_template<uint>(input, size);
}

kernel void kernel_float(cl::global_ptr<float[]> input, uint size)
{
  function_template<float>(input, size);
}
```

### <a name="S-OpenCLCXX-KernelParamsRestrictions"></a>Kernel Parameter Restrictions

The OpenCL host compiler and the OpenCL C++ kernel language device compiler can have
different requirements for i.e. type sizes, data packing and alignment, etc., therefore
the kernel parameters must meet the following requirements:

* Types passed by pointer or reference must be standard layout types.
* Types passed by value must be POD types.
* Types cannot be declared with the built-in bool scalar type, vector type or a class that
contain bool scalar or vector type fields.
* Types cannot be structures and classes with bit field members.
* Marker types must be passed by value
([Marker Types section](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#marker-types)).
* `global`, `constant`, `local` storage classes can be passed only by reference or pointer.
More details in
[Explicit address space storage classes](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#explicit-address-space-storage-classes)
section.
* Pointers and references must point to one of the following address spaces: global, local
or constant.

#### OpenCL C++ Specification References

* [OpenCL C++ Programming Language: Kernel Functions](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#kernel-functions)

### <a name="S-OpenCLCXX-GeneralRestrictions"></a>General Restrictions

The following C++14 features are not supported by OpenCL C++:

* the `dynamic_cast` operator (ISO C++ Section 5.2.7),
* type identification (ISO C++ Section 5.2.8),
* recursive function calls (ISO C++ Section 5.2.2, item 9) unless they are a compile-time constant expression,
* non-placement `new` and `delete` operators (ISO C++ Sections 5.3.4 and 5.3.5),
* `goto` statement (ISO C++ Section 6.6),
* `register` and `thread_local` storage qualifiers (ISO C++ Section 7.1.1),
* `virtual` function qualifier (ISO C++ Section 7.1.2),
* **function pointers** (ISO C++ Sections 8.3.5 and 8.5.3) **unless they are a compile-time constant expression**,
* virtual functions and abstract classes (ISO C++ Sections 10.3 and 10.4),
* exception handling (ISO C++ Section 15),
* the C++ standard library (ISO C++ Sections 17 . . . 30),
* `asm` declaration (ISO C++ Section 7.4),
* no implicit lambda to function pointer conversion (ISO C++ Section 5.1.2, item 6),
* variadic functions (ISO C99 Section 7.15, Variable arguments <stdarg.h>),
* and, like C++, OpenCL C++ does not support variable length arrays (ISO C99, Section 6.7.5).

To avoid potential confusion with the above, please note the following
features are supported in OpenCL C++:

* **All variadic templates** (ISO C++ Section 14.5.3) **including variadic function templates are supported**.

#### OpenCL C++ Specification References

* [OpenCL C++ Programming Language: Restrictions](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#opencl_cxx_restrictions)

---
## <a name="S-OpenCLCXXSTL"></a>OpenCL C++ Standard Library

OpenCL C++ does not support the C++14 standard library, but instead implements its
own standard library. It is a replacement for built-in functions provided in
OpenCL C.

##### Note
> OpenCL C++ classes and functions are NOT auto-included.

### <a name="S-OpenCLCXXSTL-NamespaceCL"></a>Namespace cl::

All class and functions provided in OpenCL C++ Standard Library are located in
namespace `cl::`.

#### OpenCL C++ Specification References

* [OpenCL C++ Standard Library](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#opencl-c-standard-library)

#### Solution

Adding a using-directive `using namespace cl;` right after including all required headers
can reduce work needed to port OpenCL C programs to OpenCL C++.

#### Examples

```cpp
#include <opencl_memory>
#include <opencl_integer> // cl::abs(gentype x)

kernel void foo(cl::global_ptr<int[]> input /* note cl:: prefix */, uint size)
{
  uint global_id = cl::get_global_id(0); // note cl:: prefix
  if(global_id < size)
  {
    using namespace cl; // no need for cl:: prefix in this scope
    input[global_id] = abs(input[global_id]);
  }
}
```

```cpp
#include <opencl_memory>
#include <opencl_integer> // cl::abs(gentype x)
using namespace cl; // No need for cl:: prefix after this using-directive

kernel void foo(global_ptr<int[]> input, uint size)
{
  uint global_id = get_global_id(0);
  if(global_id < size)
  {
    input[global_id] = abs(input[global_id]);
  }
}
```

### <a name="S-OpenCLCXXSTL-ConversionsLibrary"></a>Conversions Library

OpenCL C <code>convert&#95;<i>type</i>&lt;<i>&#95;sat</i>&gt;&lt;<i>&#95;roundingMode</i>&gt;()</code>
and <code>convert&#95;<i>typeN</i>&lt;<i>&#95;sat</i>&gt;&lt;<i>&#95;roundingMode</i>&gt;()</code> built-in
functions were replaced in OpenCL C++ with `convert_cast<>` function template. The behavior of the conversion
may be modified by one or two optional modifiers that specify saturation for out-of-range
inputs and rounding behavior.

**Rounding Modes**

```cpp
namespace cl
{
  enum class rounding_mode
  {
    rte, // Round to nearest even
    rtz, // Round toward zero
    rtp, // Round toward positive infinity
    rtn  // Round toward negative infinity
  };
}
```

##### Note
> If a rounding mode is not specified, conversions to integer type use the `rtz` (round toward zero)
rounding mode and conversions to floating-point type uses the `rte` rounding mode.

#### OpenCL C++ Specification References

* [OpenCL C++ Standard Library: Conversions Library](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#conversions-library)

#### Examples

```cpp
#include <opencl_convert>
using namespace cl; // No need for cl:: prefix after this using-directive

kernel void covert_foo_bar()
{
  int4 i { -1, 0, 1, 2 };
  float4 f { -1.5f, -0.5f, 0.5f, 1.5f};

  // Convert ints to floats using the default rounding mode (rte).
  // In OpenCL C: convert_float4_rtp(i)
  float4 f1 = convert_cast<float4>(i);

  // In OpenCL C: convert_float4_rtp(i)
  float4 f2 = convert_cast<float4, rounding_mode::rtp>(i);

  // In OpenCL C: convert_int4_sat(f)
  int4 i1 = convert_cast<int4, saturate::on>(f);

  // In OpenCL C: convert_int4_sat_rte(f)
  int4 i1 = convert_cast<int4, rounding_mode::rte, saturate::on>(f);
}
```

### <a name="S-OpenCLCXXSTL-ReinterpretingDataLibrary"></a>Reinterpreting Data Library

OpenCL C <code>as&#95;<i>type</i>()</code> and <code>as&#95;<i>typeN</i>()</code> operators used for
reinterpreting bits in a data type as another data type in OpenCL were replaced in OpenCL C++
with `TargetType as_type(InputType const&)` function template.

##### Note
> All data types described in
[Device built-in scalar data types](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#device_builtin_scalar_data_types)
and
[Device built-in vector data types](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#device_builtin_vector_data_types)
tables (except `bool` and `void`) may be also reinterpreted as another data type of the same size
using the `as_type()` function template for scalar and vector data types.

#### OpenCL C++ Specification References

* [OpenCL C++ Standard Library: Reinterpreting Data Library](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#reinterpreting-data-library)

#### Examples

```cpp
#include <opencl_reinterpret>
using namespace cl; // No need for cl:: prefix after this using-directive

kernel void reinterpret_bar_foo()
{
  float f = 1.0f;
  uint u = as_type<uint>(f); // Legal. Contains:  0x3f800000

  float4 f = float4(1.0f, 2.0f, 3.0f, 4.0f);
  // Legal. Contains:
  // int4(0x3f800000, 0x40000000, 0x40400000, 0x40800000)
  int4 i = as_type<int4>(f);

  int i;
  // Legal. Result is implementation-defined.
  short2 j = as_type<short2>(i);

  float4 f;
  // Error: result and operand have different sizes
  double4 g = as_type<double4>(f);

  float4 f;
  // Legal.
  // g.xyz will have same values as f.xyz.
  // g.w is undefined
  float3 g = as_type<float3>(f);
}
```

### <a name="S-OpenCLCXXSTL-AddressSpacesLibrary"></a>Address Spaces Library

As mentioned in [End of explicit named address spaces](#S-OpenCLCXX-EndOfExplicitNamedAddressSpaces),
in OpenCL C++ explicit named address spaces known from OpenCL C were replaced by explicit address space
storage and pointer classes.

**Explicit address space storage classes:**

* `cl::global<T> x` - allocated in global memory.
  * The global storage class can only be used to declare variables at program, function and class scope.
  * The variables at function and class scope must be declared with `static` specifier.
* `cl::local<T> x` - allocated in local memory.
  * The local storage class can only be used to declare variables at program, kernel and class scope.
  * The variables at class scope must be declared with `static` specifier.
* `cl::priv<T> x` - allocated in private memory.
  * The priv storage class cannot be used to declare variables in the program scope, with static specifier or extern specifier.
* `cl::constant<T> x` - allocated in global memory, read-only.
  * The constant storage class can only be used to declare variables at program, kernel and class scope.
  * The variables at class scope must be declared with static specifier.

**Explicit address space storage pointers classes:**

* `cl::global_ptr<T>`
* `cl::local_ptr<T>`
* `cl::private_ptr<T>`
* `cl::constant_ptr<T>`

The explicit address space pointer classes are just like pointers: they can be converted to and from pointers
with compatible address spaces, qualifiers and types. Assignment or casting between explicit pointer types of
incompatible address spaces is illegal.

All named address spaces are incompatible with all other address spaces, but local, global and private pointers
can be converted to standard C++ pointers.

#### Restrictions

[The OpenCL C++ specification](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html)
specification in subsections [3.4.5. Restrictions](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#restrictions-2)
of section [Address Spaces Library](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#address-spaces-library)
contains detailed list of restrictions with examples regarding explicit address space storage and pointer classes.
It is very important to read and understand those restrictions.

#### OpenCL C++ Specification References

* [OpenCL C++ Programming Language: Address Spaces](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#address-spaces)
* [OpenCL C++ Standard Library: Address Spaces Library](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#address-spaces-library)

#### Examples

```cpp
#include <opencl_array>
#include <opencl_memory>
#include <opencl_work_item>

int x; // Allocated in global address space
cl::global<int> y; // Allocated in global address space

cl::constant<int> z {0}; // Allocated in global address space, read-only,
                         // must be initialized

// Program scope array of 5 ints allocated in local address space
cl::local<cl::array<int, 5>> w = { 10 };

// Explicit address space class object passed by value
kernel void example_kernel(cl::global_ptr<int[]> input)
{
  cl::local<int[256]> array;

  static cl::global<int> a;
  static cl::constant<int> b {0};
}

// Explicit address space storage object passed by reference
kernel void example_kernel(cl::global<cl::array<int, 5>>& input)
{ /* ... */ }

// Explicit address space storage object passed by pointer
kernel void example_kernel(cl::global<int> * input)
{ /* ... */ }
```

##### Note
> More examples on address spaces can be found in subsections
[3.4.5. Restrictions](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#restrictions-2) and
[3.4.6. Examples](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#examples-3) of section
[Address Spaces Library](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#address-spaces-library) in
[OpenCL C++ specification](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html).

### <a name="S-OpenCLCXXSTL-MarkerTypes"></a>Marker Types

Like OpenCL C, OpenCL C++ includes special types - images, pipes.
All those types are considered marker types.
Being a marker type comes with the following set of restrictions:

* Marker types have the default constructor deleted.
* Marker types have all default copy and move assignment operators deleted.
* Marker types have address-of operator deleted.
* Marker types cannot be used in divergent control flow. It can result in undefined behavior.
* Size of marker types is undefined.

All marker types can be passed to functions only by a reference.

#### OpenCL C++ Specification References

* [OpenCL C++ Standard Library: Marker Types](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#marker-types)

#### Examples

```cpp
#include <opencl_image>
#include <opencl_work_item>
using namespace cl;

float4 bar_val(image2d<float4> img) {
    return img.read({get_global_id(0), get_global_id(1)});
}

float4 bar_ref(image2d<float4>& img) {
    return img.read({get_global_id(0), get_global_id(1)});
}

kernel void foo(image2d<float4> img)
{
    // Error: marker type cannot be passed by value
    float4 val = bar_val(img);

    // Correct, passing marker type by reference
    float4 val = bar_ref(img);
}
```

```cpp
#include <opencl_image>
#include <opencl_work_item>
using namespace cl;

float4 bar(image2d<float4> img) {
    return img.read({get_global_id(0), get_global_id(1)});
}

kernel void foo(image2d<float4> img1, image2d<float4> img2)
{
    // Error: marker type cannot be declared in the kernel
    image2d<float4> img3;

    // Error: marker type cannot be assigned
    img1 = img2;

    // Error: taking address of marker type
    image2d<float4> *imgPtr = &img1;

    // Undefined behavior: size of marker type is not defined
    size_t s = sizeof(img1);

    // Undefined behavior: divergent control flow
    float4 val = bar(get_global_id(0) ? img1: img2);
}
```

### <a name="S-OpenCLCXXSTL-ImagesAndSamplersLibrary"></a>Images and Samplers Library

Images are another part of the OpenCL that changed a lot compared to OpenCL C.
Instead of image types and built-in image read/write functions in OpenCL C++ there are
image class templates with corresponding methods. Image and sampler class templates are [marker types](#S-OpenCLCXXSTL-MarkerTypes).

#### Image types

| OpenCL C              	| OpenCL C++              	|
|-----------------------	|-------------------------	|
| image1d\_t             	| cl::image1d             	|
| image1d\_buffer\_t      	| cl::image1d\_buffer      	|
| image1d\_array\_t       	| cl::image1d\_array       	|
| image2d\_t             	| cl::image2d             	|
| image2d\_array\_t       	| cl::image2d\_array       	|
| image2d\_depth\_t       	| cl::image2d\_depth       	|
| image2d\_array\_depth\_t 	| cl::image2d\_array\_depth 	|
| image3d\_t             	| cl::image3d             	|
| sampler\_t             	| cl::sampler             	|

To instantiate image template class user has to specify image element type (which is
type returned when reading from an image, and required when writing pixel to an image),
and access mode (`cl::image_access::read` is the default access mode).

#### Image dimension

Based on the dimension of an image different methods are available. All image types have
`int width()` method, images of dimension 2 or 3 have `int height()`, 3D images have
`int depth()`, and arrayed images have one additional method - `int array_size()`.
See subsection
[Image dimension](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#image-dimension)
of OpenCL C++ Specification for more details.

#### Image element type

Depending on the type of an image different types are allowed to be specified as
image element type template parameter. Image type with invalid pixel type is ill formed.
See subsection
[Image element types](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#image-element-types)
of OpenCL C++ Specification for more details.

Image processing kernels written in OpenCL C++ can be made more readable using `.rgba` vector
component access (compared to `.xyzw` in OpenCL C).
Like `xyzw` selector, `rgba` selector works only for vector types with 4 or less elements.
See also Vector Component Access part of subsection
[Built-in Vector Data Types](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#builtin-vector-data-types)
and section
[Vector Utilities Library](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#vector-utilities-library)
of OpenCL C++ Specification.

```cpp
// OpenCL C++
kernel void openclcxx(image2d<uint4, // image element type
                              image_access::read // access mode
                             > img)
{
  uint4 color;
  // rgba selector
  color.r = 255;
  color.gb = uint2(0);
  color.a = 255;
  //...
}

// OpenCL C
kernel void openclc(read_only image2d_t img) // read_only keyword sets access mode
                                             // image element type not defined
{
  uint4 color;
  // xyzw selector
  color.x = 255;
  color.yz = (uint2)(0);
  color.w = 255;
  //...
}
```

#### Image access mode

Based on the image access mode different read and write methods are present in
the instantiated image class. See subsection
[Image access](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#image-access)
of OpenCL C++ Specification for more details.

```cpp
namespace cl
{
  enum class image_access
  {
      sample,
      read,
      write,
      read_write
  };
}
```

#### Sampler

Like in OpenCL C, in OpenCL C++ there only two ways of acquiring a sampler inside of a kernel.
One is to pass it as a kernel parameter from host using `clSetKernelArg` function,
the other is to create `cl::sampler` using `make_sampler` function in the kernel code.
The sampler objects at non-program scope must be declared with static specifier.

```cpp
template <addressing_mode A, normalized_coordinates C, filtering_mode F>
constexpr sampler make_sampler();
```

Sampler parameters and their behavior are described in subsection
[Sampler Modes](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#sampler-modes)
of OpenCL C++ Specification.

#### OpenCL C++ Specification References

* [OpenCL C++ Standard Library: Images and Samplers Library](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#images-and-samplers-library)
* [OpenCL C++ Standard Library: Marker Types](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#marker-types)

#### Examples

```cpp
// OpenCL C++
#include <opencl_image>
#include <opencl_work_item>
using namespace cl;

using my_image1d_type = image1d<float4, // image element type
                                image_access::write>; // access mode

using my_image2d_type = image2d<float4>; // access mode is image_access::read

kernel void openclcxx(my_image1d_type img1d, my_image2d_type img2d)
{
    const int  coords1d(get_global_id(0));
    const int2 coords2d(get_global_id(0), get_global_id(1));

    float4 val1d(0.0f);
    // 1) write() is enabled because the access mode of my_image1d_type
    //    is image_access::write
    // 2) write() takes int value as pixel coordinates because my_image1d_type
    //    is a 1d image type
    // 3) write() takes float4 value as pixel value because float4 is the image
    //    element type of my_image1d_type
    img1d.write(coords1d, val1d);

    // 1) read() is enabled because the access mode of my_image2d_type
    //    is image_access::read
    // 2) read() takes int2 as an input argument because my_image2d_type
    //    is a 2d image type
    // 3) read() returns float4 because float4 is the image element type
    //    of my_image2d_type
    float4 val2d = img2d.read(coords2d);
}
```

```cpp
// OpenCL C
kernel void openclc(write_only image1d_t img1d, // write_only keyword sets access mode
                    read_only  image2d_t img2d) // read_only keyword sets access mode
{
    const int  coords1d = get_global_id(0);
    const int2 coords2d = (int2)(get_global_id(0), get_global_id(1));

    float4 val1d = (float4)(0.0f);
    write_imagef(img1d, coords1d, val1d);

    // float4 read_imagef(image2d_t, int2) function is used to
    // read from img 2d image.
    float4 val2d = read_imagef(img2d, coords2d);
}
```

### <a name="S-OpenCLCXXSTL-PipesLibrary"></a>Pipes Library

In OpenCL C++ `pipe` keyword was replaced with `cl::pipe` class template.
Reserve operations return `cl::pipe::reservation` object, instead of returning
reservation id of type `reserve_id_t`.

All `pipe`s-related function were moved to `cl::pipe` or `reservation` as
their methods.

#### Pipe storage

OpenCL C++ introduces new pipe-related type - `cl::pipe_storage` class template.
It enables programmers to create `cl::pipe` objects in an OpenCL program without
need to create `cl_pipe` on host using API. `cl::pipe_storage` class template has
two template parameters: `T` - element type, and `N` - the maximum number of packets
which can be held by an object.

##### Note
One kernel can have only one pipe accessor (`cl::pipe` object) associated with
one `cl::pipe_storage` object.

#### Requirements and Restictions

`cl::pipe::reservation`, `cl::pipe_storage` and `cl::pipe` are marker types.
However, they also have additional sets of requirements and restictions beyond
those specified in
[Market Types](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#marker-types)
section. The most important are:

* The element type `T` of `pipe` and `pipe_storage` class templates
must be a POD type i.e. satisfy `is_pod<T>::value == true`.
* A kernel cannot read from and write to the same pipe object.
* Variables of type `pipe_storage` can only be declared at program scope or
with the `static` specifier.
* Variables of type `pipe` created from `pipe_storage` can only be declared
inside a kernel function at kernel scope.
* The `reservation`, `pipe_storage`, and `pipe` types cannot be used as a class or
union field, a pointer type, an array or the return type of a function.
* The `reservation`, `pipe_storage`, and `pipe` types cannot be used with the
`global`, `local`, `priv` and `constant` address space storage classes.

The full lists of requirements and restictions can be found in subsections
[Requirements](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#requirements) and
[Restrictions](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#restrictions-5)
of Pipe Library section in OpenCL C++ Specification.

#### OpenCL C++ Specification References

* [OpenCL C++ Standard Library: Pipes Library](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#pipes-library)
  *  [Requirements](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#requirements)
  *  [Restrictions](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#restrictions-5)
* [OpenCL C++ Standard Library: Marker Types](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#marker-types)

#### Examples

Reading from and writing to a pipe:

```cpp
// OpenCL C++
#include <opencl_pipe>

kernel void foobar(cl::pipe<int /* type */, cl::pipe_access::write /* access mode */> wp,
                   cl::pipe<int /* access mode defaults to read */> rp)
{
  int val;
  // ...
  // write() method is enabled only for pipes with
  // pipe_access::write access mode
  if(wp.write(val)) { // val passed by const reference
      // ...
  }

  // read() method is enabled only for pipes with
  // pipe_access::read access mode
  if(rp.read(val)) { // val passed by reference
      // ...
  }
}
```

```cpp
// OpenCL C
kernel void foobar(write_only /* access mode */ pipe /* keyword */ int /* type */ wp,
                   read_only  /* access mode */ pipe /* keyword */ int /* type */ rp)
{
  int val;
  // ...

  // In OpenCL write_pipe(...) and read_pipe(...) operations
  // returns 0 when write/read is successful, and a negative
  // value otherwise
  if(write_pipe(p, &val) == 0) {
      // ...
  }

  if(read_pipe(p, &val) == 0) {
      // ...
  }
}
```

```cpp
// OpenCL C++
#include <opencl_pipe>

kernel void foobar(cl::pipe<int> p)
{
  int val;
  // cl::pipe<int, cl::pipe_access::read>::reservation<memory_scope_work_item>
  auto r = p.reserve(3);
  // ...
  // read() method is available because pipe p is in
  // pipe_access::read access mode
  if(r.read(2, val)) {
    // ...
  }
  r.commit();
}
```

Making and using a reservation:

```cpp
// OpenCL C
kernel void foobar(read_only pipe int p)
{
  int val;
  reserve_id_t rid = reserve_read_pipe(p, 3);
  // ...
  if(read_pipe(p, rid, 2, &val)) {
      // ...
  }
  commit_read_pipe(p, rid);
}
```

```cpp
// OpenCL C++
#include <opencl_pipe>

kernel void foobar(cl::pipe<int> p)
{
  int val;
  // cl::pipe<int, cl::pipe_access::read>::reservation<memory_scope_work_item>
  auto r = p.reserve(3);
  // ...
  // read() method is available because pipe p is in
  // pipe_access::read access mode
  if(r.read(2, val)) {
    // ...
  }
  r.commit();
}
```

Using `pipe_storage`:

```cpp
// OpenCL C++
#include <opencl_pipe>

cl::pipe_storage <int, 1337> my_pipe;

kernel void reader()
{
  auto p = my_pipe.get<cl::pipe_access::read>();
  // ...
  p.read(...);
  // ...
}

kernel void writer()
{
  auto p = my_pipe.get<cl::pipe_access::write>();
  // ...
  p.write(...);
  // ...
}

kernel void error_kernel()
{
  auto p1 = my_pipe.get<cl::pipe_access::write>();
  // Error, one kernel can have only one pipe accessor
  // (cl::pipe object) associated with one cl::pipe_storage object.
  auto p2 = my_pipe.get<cl::pipe_access::read>();
  // ...
}
```

### <a name="S-OpenCLCXXSTL-DeviceEnqueueLibrary"></a>Device Enqueue Library

When it comes to enqueuing a kernel without host interaction, the biggest difference between
OpenCL C and OpenCL C++ is that in OpenCL C++ enqueued kernel can be a lambda expression or
a function, whereas in OpenCL C it is defined using block syntax.

All functions except function which returns default device queue and kernel query functions
were moved to appropriate classes as their methods.
See [Header <opencl_device_queue> Synopsis](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#header-opencl_device_queue-synopsis)
subsections of OpenCL C++ specification.

#### Device Queue

In OpenCL C++ `cl::device_queue` class represents device queue (`queue_t` in OpenCL C).
`cl::device_queue` is a marker type (see [Marker Types](#S-OpenCLCXXSTL-MarkerTypes)).

| OpenCL C              	| OpenCL C++              	|
|-----------------------	|-------------------------	|
| queue\_t             	  | cl::device\_queue         |

```cpp
namespace cl
{
  struct device_queue: marker_type
  {
    // ...

    template <class Fun, class... Args>
    enqueue_status enqueue_kernel(enqueue_policy flag,
                                  const ndrange &ndrange,
                                  Fun fun,
                                  Args... args) noexcept;

    // In OpenCL C:
    // int enqueue_kernel(queue_t queue,
    //                    kernel_enqueue_flags_t flags,
    //                    const ndrange_t ndrange,
    //                    void (^block)(local void *, ...),
    //                    uint size0, ...);

    // ...
  };
}
```

##### Note
>`args` are the arguments that will be passed to `fun` when kernel will be enqueued with
the exception for `local_ptr` parameters. For local pointers user must supply the size of
local memory that will be allocated using <code>local\_ptr<T>::size\_type{<i>num_elements</i>}</code>.
In OpenCL C user has to pass `uint` value for a corresponding local pointer, which specifies
the size of a local memory accessible using that local pointer.

#### Event

In OpenCL C++ `cl::event` class represents device-side event (`clk_event_t` in OpenCL C).

| OpenCL C              	| OpenCL C++              	|
|-----------------------	|-------------------------	|
| clk\_event\_t      	    | cl::event      	          |

`cl::event` has the same possible states as `clk_event_t`, however in OpenCL C++ error is
not represented by any negative value, but rather by `cl::event_status::error` enum.

| OpenCL C              	| OpenCL C++              	| Description              	|
|-----------------------	|-------------------------	|-------------------------	|
| CL\_SUBMITTED     	| cl::event\_status::submitted      	| Initial status of a user event |
| CL\_COMPLETE      	| cl::event\_status::complete      	| |
| Any negative integer value      	| cl::event\_status::error      	| Status indicating an error |

See [Event Class Methods](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#event-class-methods) and
[Event Status](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#event-status)
subsections of OpenCL C++ specification.

#### Enqueue Policy

Available enqueue policies did not changed compared to OpenCL C.
In OpenCL C enqueue policy type was `kernel_enqueue_flags_t` enum, in OpenCL C++ it is
`cl::enqueue_policy` enum class.

| OpenCL C              	| OpenCL C++              	|
|-----------------------	|-------------------------	|
| CLK_ENQUEUE_FLAGS_NO_WAIT     	| cl::enqueue\_polic::no\_wait     	|
| CLK_ENQUEUE_FLAGS_WAIT_KERNEL       	| cl::enqueue\_polic::wait\_kernel      	|
| CLK_ENQUEUE_FLAGS_WAIT_WORK_GROUP      	| cl::enqueue\_polic::wait\_work\_group      	|

See [Enqueue Policy](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#enqueue-policy)
subsection of OpenCL C++ specification.

#### Requirements

Functor and lambda objects passed to `enqueue_kernel()` method of device queue has to follow
specific restrictions:

* It has to be trivially copyable.
* It has to be trivially copy constructible.
* It has to be trivially destructible.

Code enqueuing function objects that do not meet this criteria is ill-formed.

#### OpenCL C++ Specification References

* [OpenCL C++ Standard Library: Device Enqueue Library](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#device-enqueue-library)
  * [Restrictions](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#restrictions-6)
  * [Examples](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#examples-7)
* [OpenCL C++ Standard Library: Marker Types](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#marker-types)

#### Examples

Block syntax vs. lambda expression:

```cpp
// OpenCL C++
#include <opencl_device_queue>
#include <opencl_memory>

kernel void my_func(cl::global_ptr<int> a, cl::global_ptr<int> b, cl::global_ptr<int> c)
{
  // ...
  auto dq = cl::get_default_device_queue();
  dq.enqueue_kernel(
    cl::enqueue_polic::no_wait,
    cl::ndrange({10, 10}),
    [=](){          //
      *a = *b + *c; // Lambda expression
    }               //
  );
  // ...
}
```

```cpp
// OpenCL C
kernel void my_func(global int *a, global int *b, global int *c)
{
  // ...
  enqueue_kernel(
    get_default_queue(),
    CLK_ENQUEUE_FLAGS_NO_WAIT,
    ndrange_2D(1, 1),
    ^{              //
      *a = *b + *c; // Block syntax
    }               //
  );
  // ...
}
```

Enqueuing a functor:

```cpp
// OpenCL C++
#include <opencl_device_queue>
#include <opencl_memory>

struct my_functor {
    void operator ()(cl::local_ptr<ushort16[]> p, int x) const
    { /* ... */ }
};

kernel void my_func(cl::device_queue q)
{
  // ...
  my_functor f;
  dq.enqueue_kernel(
    cl::enqueue_polic::no_wait,
    cl::ndrange(1),
    f, // functor
    cl::local_ptr<ushort16[]>::size_type{10}, // define size of p
    2 // x
  );
  // ...
}
```

### <a name="S-OpenCLCXXSTL-RelationalFunctions"></a>Relational Functions

In OpenCL C++ there were significant changes in signatures and/or behaviour of
built-in relational functions. This is because OpenCL C++ introduces
<code>bool<i>N</i></code> type which can replace <code>int<i>N</i></code> as a type
returned by relational functions.

#### `all()` and `any()`

In OpenCL C:
```cpp
// igentype can be char, charN, short, shortN, int, intN, long, and longN
int any (igentype x);
int all (igentype x);
```
>`any()` returns 1 if **the most significant bit** in any component of `x` is set; otherwise returns 0.

>`all()` returns 1 if **the most significant bit** in all components of `x` is set; otherwise returns 0.

In OpenCL C++:

```cpp
bool any(booln t);
bool all(booln t);
```
>`any()` returns `true` if any component of `t` is `true`; otherwise returns `false`.

>`all()` returns `true` if all components of `t` are `true`; otherwise returns `false`.

#### `select()`

In OpenCL C:
```cpp
// igentype can be char, charN, short, shortN, int, intN, long, and longN
// ugentype can be uchar, ucharN, ushort, ushortN, uint, uintN, ulong, and ulongN
gentype select (gentype a, gentype b, igentype c);
gentype select (gentype a, gentype b, ugentype c);
```
> For each component of a vector type, `result[i] = if MSB of c[i] is set ? b[i] : a[i]`.

> For scalar type, `result = c ? b : a`.

> `igentype` and `ugentype` must have the same number of elements and bits as `gentype`.

> NOTE: The above definition means that the behavior of select and the ternary operator
for vector and scalar types is dependent on different interpretations of the bit pattern of `c`.

In OpenCL C++ `select()` is less confusing:
```cpp
gentype select(gentype a, gentype b, booln c);
```
> For each component of a vector type, `result[i] = c[i] ? b[i] : a[i]`.

> For a scalar type, `result = c ? b : a`.

> <code>bool<i>N</i></code> must have the same number of elements as gentype.

#### OpenCL C++ Specification References

* [OpenCL C++ Standard Library: Relational Functions](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#relational-functions)
* [OpenCL C++ Programming Language: Expressions](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#expressions)

#### Examples

```cpp
// OpenCL C++
#include <opencl_relational>
kernel void foobar()
{
  bool b1 = isequal(1.0f, 1.0f); // true
  bool b2 = isequal(1.0, 2.0); // false

  bool2 b3 = isequal(float2(1.0f), float2(1.0f)); // { true, true }
  bool2 b4 = isequal(double2(1.0), double2(2.0)); // { false, false }

  bool2 b5 = { true, false };
  auto b6 = all(b3); // false
  auto b7 = any(b3); // true

  bool2 c { true, false };
  float2 a {  1.0f,  1.0f };
  float2 b { -1.0f, -1.0f };
  auto r1 = select(a, b, c); // { -1.0f, 1.0f }

  auto r2 = select(1.0f, 2.0f, false); // 1.0f
}
```

```cpp
// OpenCL C
kernel void foobar()
{
  // Note: in integer value -1 MSB is set to 1

  int b1 = isequal(1.0f, 1.0f); // 1 (true)
  long b2 = isequal(1.0, 2.0); // 0 (false)

  int2 b3 = isequal((float2)(1.0f), (float2)(1.0f)); // { -1, -1 } ({ true, true })
  long2 b4 = isequal((double2)(1.0), (double2)(2.0)); // { 0, 0 }  ({ false, false })

  int b5 = all( (int2)(-1, 10) ); // 0
  int b6 = all( (int2)(-1, -1) ); // 1

  int b7 = any( (int2)(-1, 0) ); // 1
  int b8 = any( (int2)(1, 1) ); // 0

  int2 c = (int2)(-1, 1);
  float2 a = (float2)(1.0f, 1.0f);
  float2 b = (float2)(-1.0f, -1.0f);
  float2 r1 = select(a, b, c); // { -1.0f, 1.0f }

  float r2 = select(1.0f, 2.0f, -1); // 2.0f
  float r3 = select(1.0f, 2.0f,  1); // 1.0f
  float r4 = select(1.0f, 2.0f,  0); // 1.0f
}
```

### <a name="S-OpenCLCXXSTL-VectorDataLoadandStoreFunctions"></a>Vector Data Load and Store Functions

In OpenCL C++ vector data load and store functions were greatly simplified compared to OpenCL: instead of
39 different functions, now there are just 9 function templates. The requirements and the behaviours of
functions have not be changed. Also arguments and their order was not changed.

| OpenCL C              	| OpenCL C++              	|
|-----------------------	|-------------------------	|
| <code>gentype<i>N</i> vload<i>N</i></code> | `template <size_t N, class T> make_vector_t<T, N> vload` |
| <code>void vstore<i>N</i>(...)</code> | `template <class T> void vstore(…, vector_element_t<T>* p)` |
| <code>float<i>N</i> vload_half\[<i>N</i>\]</code> | `template <size_t N> make_vector_t<float, N> vload_half` |
| <code>void vstore_half[<i>N</i>]\[<i>\_rounding\_mode</i>\]</code> | `template <rounding_mode R, class Type> void vstore_half(…, half* p)` |
| <code>float<i>N</i> vloada_half<i>N</i></code> | `template <size_t N> make_vector_t<float, N> vloada_half` |
| <code>void vstore_half<i>N</i>\[<i>\_rounding\_mode</i>\]</code> | `template <rounding_mode R, class T> void vstorea_half(…, half* p)` |

Read [Header <opencl_vector_load_store> Synopsis](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#header-opencl_vector_load_store)
subsection of [Vector Data Load and Store Functions](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#vector-data-load-and-store-functions)
section to see vector data load and store function templates declarations.

#### OpenCL C++ Specification References

* [OpenCL C++ Standard Library: Vector Data Load and Store Functions](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#vector-data-load-and-store-functions)

#### Examples

`vload` and `vstore`:

```cpp
// OpenCL C++
#include <opencl_vector_load_store>
using namespace cl;

kernel void foobar(float * fptr, const constant_ptr<half> hptr)
{
  auto f4 = vload<4>(0, fptr); // reads from (fptr + (0 * 4)), float4 returned
  auto f2 = vload<2>(2, fptr); // reads from (fptr + (2 * 2)), float2 returned

#ifdef cl_khr_fp16 // cl_khr_fp16 must be defined and supported
  auto h8 = vload<8>(0, hptr); // reads from (hptr + (0 * 8)), half8 returned
#endif

  vstore(float4{ 1, 2, 3, 4}, 0, fptr); // float4 stored at (fptr + (0 * 4))
  vstore(f2, 2, fptr); // float2 stored at (fptr + (2 * 2))
}
```

```cpp
// OpenCL C
kernel void foobar(float * fptr, const constant half * hptr)
{
  float4 f4 = vload4(0, fptr); // reads from (fptr + (0 * 4)), float4 returned
  float2 f2 = vload2(2, fptr); // reads from (fptr + (2 * 2)), float2 returned

#ifdef cl_khr_fp16 // cl_khr_fp16 must be defined and supported
  half8 h8 = vload8(0, hptr); // reads from (hptr + (0 * 8)), half8 returned
#endif

  vstore4(f4, 0, fptr); // float4 stored at (fptr + (0 * 4))
  vstore2(f2, 2, fptr); // float2 stored at (fptr + (2 * 2))
}
```

`vload_half`, `vstore_half`, `vloada_half`, and `vstorea_half`:

```cpp
// OpenCL C++
#include <opencl_vector_load_store>
using namespace cl;

kernel void foobar_half(half * hptr)
{
  // half vload
  auto f4 = vload_half<4>(0, hptr); // reads from (hptr + (0 * 4)), float4 returned
  auto f3 = vload_half<3>(0, hptr); // reads from (hptr + (0 * 3)), float3 returned

  // half array vload
  auto f4a = vloada_half<4>(0, hptr); // reads from (hptr + (0 * 4)), float4 returned
  auto f3a = vloada_half<3>(0, hptr); // reads from (hptr + (0 * 4)), float3 returned

  // half vstore
  vstore_half(f3, 0, hptr); // float3 stored at (hptr + (0 * 3)),
                            // rounded to nearest even (rounding_mode::rte)
  vstore_half<rounding_mode::rtz>(f4, 0, hptr); // float4 stored at (hptr + (0 * 4)),
                                                // rounded toward zero
  // half array vstore
  vstorea_half(f3a, 0, hptr); // float3 stored at (hptr + (0 * 4))
                              // rounded to nearest even (rounding_mode::rte)
  vstorea_half<rounding_mode::rtz>(f4a, 0, hptr); // float4 stored at (hptr + (0 * 4))
                                                  // rounded toward zero
}
```

```cpp
// OpenCL C
kernel void foobar_half(half * hptr)
{
  // half vload
  float4 f4 = vload_half4(0, hptr); // reads from (hptr + (0 * 4)), float4 returned
  float3 f3 = vload_half3(0, hptr); // reads from (hptr + (0 * 3)), float3 returned

  // half array vload
  float4 f4a = vloada_half4(0, hptr); // reads from (hptr + (0 * 4)), float4 returned
  float3 f3a = vloada_half3(0, hptr); // reads from (hptr + (0 * 4)), float3 returned

  // half vstore
  vstore_half3(f3, 0, hptr); // float3 stored at (hptr + (0 * 3)),
                             // rounded to nearest even
  vstore_half4_rtz(f4, 0, hptr); // float4 stored at (hptr + (0 * 4)),
                                 // rounded toward zero

  // half array vstore
  vstorea_half3(f3a, 0, hptr); // float3 stored at (hptr + (0 * 4))
                               // rounded to nearest even
  vstorea_half4_rtz(f4a, 0, hptr); // float4 stored at (hptr + (0 * 4))
                                   // rounded toward zero
}

```

### <a name="S-OpenCLCXXSTL-AtomicOperationsLibrary"></a>Atomic Operations Library

OpenCL C atomic operation are based on C11 atomics. In OpenCL C++ atomics are based on
C++14 atomics and synchronization operations.
Section [Atomic Operations Library](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#atomic-operations-library)
of OpenCL C++ presents synopsis of the atomics library and differences from C++14 specification.

Because atomic functions in OpenCL C and OpenCL C++ have virtually the same argument lists adding
`using namespace cl;` can Significantly speed up porting kernels to OpenCL C++.

#### Atomic types

In OpenCL C++ different OpenCL C atomic types like `atomic_int`, `atomic_float` were replaced with one class
template `atomic<T>`, however, for supported types proper type alias are declared
(for example: `using atomic_int = atomic<int>;`).

* There are explicit specializations for integral types. Each of these specializations provides set of extra
operators suitable for integral types.
* There is an explicit specialization of the atomic template for pointer types.
* All atomic classes have deleted copy constructor and deleted copy assignment operators.
* 64-bit atomic types require `cl_khr_int64_base_atomics` and `cl_khr_int64_extended_atomics` extensions
and `atomic<double>` in addition requires `cl_khr_fp64`.

#### Restrictions

* The generic `atomic<T>` class template is only available if `T` is `int`, `uint`, `long`,
 `ulong`, `float`, `double`, `intptr_t`, `uintptr_t`, `size_t`, `ptrdiff_t`.
* The atomic data types cannot be declared inside a kernel or non-kernel function unless they are declared as `static` keyword or in `local<T>` and `global<T>` containers. See examples.
* The atomic operations on the private memory can result in undefined behavior.
* `memory_order_consume` from C++14 is not supported by OpenCL C++.

Full list of restrictions can be found in subsection
[Restrictions](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#restrictions-3) of section
[Atomic Operations Library](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#atomic-operations-library)
in OpenCL C++ specification.

#### OpenCL C++ Specification References

* [OpenCL C++ Standard Library: Atomic Operations Library](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#atomic-operations-librarys)

#### Examples

```cpp
// OpenCL C++
#include <opencl_memory>
#include <opencl_atomic>
using namespace cl;

atomic_int a; // OK: program scope atomic in the global memory
              // atomic_int is alias for atomic<int>
local<atomic<int>> b(1); // OK: program scope atomic in the local memory
                         // Initialized to 1. The initialization is not atomic.
global<atomic<int>> c = ATOMIC_VAR_INIT(2); // OK: program scope atomic in the global memory
                                            // Initialized to 2. The initialization is not atomic.

kernel void foo()
{
    static global<atomic<int>> d; // OK: atomic in the global memory
    static atomic<int> e; // OK: atomic in the global memory
    local<atomic<int>> f; // OK: atomic in the local memory

    atomic<global<int>> g; // Error: class members cannot be
                           //        in address space

    atomic<int> h; // undefined behavior

    atomic_init(&a, 123); // Initialize a to 123. The initialization is not atomic.
}
```

```cpp
// OpenCL C+
global atomic_int a; // OK: program scope atomic in the global memory
local  atomic_int b; // Error: program scope local variables not suppoerted in OpenCL C
global atomic_int c = ATOMIC_VAR_INIT(2); // OK: program scope atomic in the global memory
                                          // Initialized to 2. The initialization is not atomic.

kernel void foo()
{
    static global atomic_int d; // OK: atomic in the global memory
    static atomic_int e; // OK: atomic in the global memory
    local atomic_int f; // OK: atomic in the local memory

    atomic_int h; // undefined behavior

    atomic_init(&a, 123); // Initialize a to 123. The initialization is not atomic.
}
```

---
## <a name="S-OpenCLCXXCompilation"></a>OpenCL C++ Compilation Process

OpenCL C++ kernel language can not be consumed by `clCreateProgramWithSource()` API function, which
is used to create a program from OpenCL C source. OpenCL C++ source first have to be compiled
to SPIR-V 1.2 binary, which can later be passed to `clCreateProgramWithIL()` to create an OpenCL program.
After that program can be build with `clBuildProgram()`.

### <a name="S-OpenCLCXXCompilationToSPIRV"></a>OpenCL C++ Compilation to SPIR-V

To compile OpenCL C++ kernel language to SPIR-V user have to use compiler that is not a part
of OpenCL framework. The Khronos Group provides reference
[offline compiler based on Clang 3.6](https://github.com/KhronosGroup/SPIR/tree/spirv-1.1)
and an implementation of OpenCL C++ Standard Library called [libclcxx](https://github.com/KhronosGroup/libclcxx).

#### Preprocessor options

Every preprocessor option that would normally be specified in `clBuildProgram()`, for OpenCL C++ must
be passed when it is being compiled to SPIR-V.

```
-D name
```

Predefine name as a macro, with definition 1.

```
-D name=definition
```

The contents of definition are tokenized and processed as if they appeared during translation phase
three in a `#define` directive.
In particular, the definition will be truncated by embedded newline characters.

#### Other compilation options

Some feature-related options must be specified during compilation to SPIR-V:

* `-cl-fp16-enable` - enables full half data type support and defines `cl_khr_fp16` macro. Disabled by default.
* `-cl-fp64-enable` - enables full double data type support and defines `cl_khr_fp64` macro. Disabled by default.
* `-cl-zero-init-local-mem-vars` -  enables software zero-initialization of variables allocated in local memory.

### <a name="S-OpenCLCXXCompilationBuildSPIRV"></a> Building program created from SPIR-V
When an OpenCL program created using `clCreateProgramWithIL()` is compiled (`clBuildProgram()`) not
all build options are allowed. They have to be passed when compiling to SPIR-V. Otherwise, there is
no difference between building program created from SPIR-V and program created from OpenCL C source.
Which options are ignored and which not is described in
[OpenCL 2.2 API Specification](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2.html#_compiler_options).

#### OpenCL C++ Specification and OpenCL 2.2 API References

* [OpenCL C++ Specification: Compiler options](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html#compiler_options)
* [OpenCL 2.2 Specification: Compiler Options](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2.html#_compiler_options)


# <a name="S-Bibliography"></a>Bibliography

### OpenCL Specifications

* [The OpenCL C++ 1.0 Specification](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.pdf)
 ([HTML](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.html))
* [The OpenCL 2.2 API Specification](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2.pdf)
 ([HTML](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2.html))
* [The OpenCL 2.2 Extension Specification](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-extension.pdf)
 ([HTML](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-extension.html))
* [OpenCL 2.2 SPIR-V Environment Specification](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-environment.pdf)
 ([HTML](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-environment.html))
* [The OpenCL C 2.0 Language Specification](https://www.khronos.org/registry/OpenCL/specs/opencl-2.0-openclc.pdf)
* [The OpenCL 2.1 API Specification](https://www.khronos.org/registry/OpenCL/specs/opencl-2.1.pdf)

### OpenCL Reference Pages

* The OpenCL 2.2 Reference Page (not published yet)
* [The OpenCL 2.1 Reference Page](http://www.khronos.org/registry/cl/sdk/2.1/docs/man/xhtml/)

### OpenCL Headers

* [OpenCL 2.2 Headers](https://github.com/KhronosGroup/OpenCL-Headers/tree/master/opencl22)
* [OpenCL 2.1 Headers](https://github.com/KhronosGroup/OpenCL-Headers/tree/master/opencl21)

### Other

* [Khronos OpenCL Registry](https://www.khronos.org/registry/OpenCL/)
([GitHub](https://github.com/KhronosGroup/OpenCL-Registry))
* [OpenCL 2.2 Release Note](https://www.khronos.org/news/press/khronos-releases-opencl-2.2-with-spir-v-1.2)
* Michael Wong, Adam Stanski, Maria Rovatsou, Ruyman Reyes, Ben Gaster, and Bartok Sochaski. 2016.
C++ for OpenCL Workshop, IWOCL 2016. In Proceedings of the 4th International Workshop
on OpenCL (IWOCL '16).
  * [Dive into OpenCL C++](http://www.iwocl.org/wp-content/uploads/iwocl-2016-dive-into-opencl-c.pdf)
  * [OpenCL C++ kernel language](http://www.iwocl.org/wp-content/uploads/iwcol-2016-opencl-ckernel-language.pdf)

***

OpenCL and the OpenCL logo are trademarks of Apple Inc. used by permission by Khronos.
Other names are for informational purposes only and may be trademarks of their respective owners.
