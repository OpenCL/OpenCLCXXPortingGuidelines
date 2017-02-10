# <a name="title"></a>OpenCL C to OpenCL C++ Porting Guidelines

February 10, 2017

Editors:

* [Jakub Szuppe, StreamComputing](https://streamcomputing.eu/)
* [Your name](#)

This document is a set of guidelines for developers who know OpeCL C and plan to
port their kernels to OpenCL C++, and therefore they need to know the main
differences between those two kernel languages.
The focus is not on highlighting all the differences, but rather on exposing
and explaining those that are the most important, and those that may cause
hard-to-detect bugs when porting from OpenCL C to OpenCL C++.

Comments and suggestions for improvements are most welcome.

#### List of discussed issues:

* [OpenCL C++ Programming Language](#C-OpenCLCXX):
  * [OpenCL C vector literals](#C-OpenCLCXX-S-VectorLiterals)
  * [Kernel Function Restrictions](#C-OpenCLCXX-S-KernelRestrictions)
  * [Kernel Parameter Restrictions](#C-OpenCLCXX-S-KernelParamsRestrictions)
  * [General Restrictions](#C-OpenCLCXX-S-GeneralRestrictions)
* [OpenCL C++ Standard Library](#C-OpenCLCXXSTL):
  * [Namespace cl::](#C-OpenCLCXXSTL-S-NamespaceCL)
  * TODO
* [OpenCL C++ Compilation Process](#C-OpenCLCXXCompilation):
  * TODO

---
## <a name="C-OpenCLCXX"></a>OpenCL C++ Programming Language

### <a name="C-OpenCLCXX-S-VectorLiterals"></a>OpenCL C vector literals

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
> Vector literals in OpenCL C++ are NOT evaluated as user might expect,
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

### <a name="C-OpenCLCXX-S-KernelRestrictions"></a>Kernel Function Restrictions

TODO

### <a name="C-OpenCLCXX-S-KernelParamsRestrictions"></a>Kernel Parameter Restrictions

TODO

### <a name="C-OpenCLCXX-S-GeneralRestrictions"></a>General Restrictions

TODO

---
## <a name="C-OpenCLCXXSTL"></a>OpenCL C++ Standard Library

OpenCL C++ does not support the C++14 standard library, but instead implements its
own standard library. It is a replacement for built-in functions provided in
OpenCL C.

##### Note
> OpenCL C++ classes and functions are NOT auto-included.

### <a name="C-OpenCLCXXSTL-S-NamespaceCL"></a>Namespace cl::

All class and funtions provided in OpenCL C++ Standard Library are located in
namespace `cl::`.

#### Examples

A using-directive `using namespace cl;` can be used to reduce work need to port
OpenCL C programs to OpenCL C++. However, it is important to remember to include
required headers.

```cpp
#include <opencl_memory>
#include <opencl_integer> // cl::abs(gentype x)

kernel void foo(cl::global_ptr<int[]> input, uint size)
{
  uint global_id = cl::get_global_id(0); // note cl:: prefix
  if(global_id < size)
  {
    using namespace cl; // no need for cl:: prefix in this scope
    input[global_id] = abs(input[global_id]);
  }
}
```

### TODO

* bool and booln types are use in many function instead of int, intn, long, longn
types
* all, any and select functions are slightly different because of using booln type
* (type traits) I think we can mention useful tools for vectors like:
make_vector<T, N>, scalar_type<T> etc.
* Atomics are now classes, not set of functions (but seem to be compatible
with OpenCL C via typedefs)
* pipe is not a keyword anymore, pips is a class now
* half wrapper?

---
## <a name="C-OpenCLCXXCompilation"></a>OpenCL C++ Compilation Process

### TODO

* now there are two compilers: front-compiler (OpenCL C++ to SPIR-V) and
back-compiler (SPIR-V to device machine code)
