# <a name="title"></a>OpenCL C to OpenCL C++ Porting Guidelines

February 14, 2017

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

#### [Differences](#S-Differences):

* [OpenCL C++ Programming Language](#S-OpenCLCXX):
  * [OpenCL C vector literals](#S-OpenCLCXX-VectorLiterals)
  * [Kernel Function Restrictions](#S-OpenCLCXX-KernelRestrictions)
  * [Kernel Parameter Restrictions](#S-OpenCLCXX-KernelParamsRestrictions)
  * [General Restrictions](#S-OpenCLCXX-GeneralRestrictions)
  * [End of explicit named address spaces](#S-OpenCLCXX-EndOfExplicitNamedAddressSpaces)
* [OpenCL C++ Standard Library](#S-OpenCLCXXSTL):
  * [Namespace cl::](#S-OpenCLCXXSTL-NamespaceCL)
  * [Address Spaces Library](#S-OpenCLCXXSTL-AddressSpacesLibrary)
  * TODO
* [OpenCL C++ Compilation Process](#S-OpenCLCXXCompilation):
  * TODO

#### [Bibliography](#S-Bibliography)

# <a name="S-Differences"></a>Differences

## <a name="S-OpenCLCXX"></a>OpenCL C++ Programming Language

### <a name="S-OpenCLCXX-VectorLiterals"></a>OpenCL C vector literals

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

### <a name="S-OpenCLCXX-EndOfExplicitNamedAddressSpaces"></a>End of explicit named address spaces

[OpenCL C++ 1.0 Specification in Address Spaces section](LINK_TO_OPENCLCXX_SPEC_HTML#address-spaces) says:
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

It is still possible for OpenCL C++ compile to deduce an address space based on the scope where
an object is declared:

* If a variable is declared in program scope, with `static` or `extern` specifier and the standard
library storage class
(see [Explicit address space storage classes](LINK_TO_OPENCLCXX_SPEC_HTML#explicit-address-space-storage-classes) section)
is not used, the variable is allocated in the global memory of a device.
* If a variable is declared in function scope, without static specifier and the standard library storage class
(see [Explicit address space storage classes](LINK_TO_OPENCLCXX_SPEC_HTML#explicit-address-space-storage-classes) section)
is not used, the variable is allocated in the private memory of a device.

#### OpenCL C++ Specification References

* [OpenCL C++ Programming Language: Address Spaces](LINK_TO_OPENCLCXX_SPEC_HTML#address-spaces)
* [OpenCL C++ Standard Library: Address Spaces Library](LINK_TO_OPENCLCXX_SPEC_HTML#address-spaces-library)

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
> More examples on address spaces can be found in subsections [3.4.5. Restrictions](LINK_TO_OPENCLCXX_SPEC_HTML#restrictions-2) and [3.4.6. Examples](LINK_TO_OPENCLCXX_SPEC_HTML#examples-3) of section [Address Spaces Library](LINK_TO_OPENCLCXX_SPEC_HTML#address-spaces-library) in [OpenCL C++ specification](LINK_TO_OPENCLCXX_SPEC_HTML).

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

* [OpenCL C++ Programming Language: Kernel Functions](LINK_TO_OPENCLCXX_SPEC_HTML#kernel-functions)

#### Examples, bad

```cpp
// A kernel function cannot be template function.
template<class T>
kernel void example_kernel(cl::global_ptr<T[]> input, uint size)
{
  // ...
}

// A kernel function cannot have parameters specified with default values.
kernel void foo(cl::global_ptr<uint[]> input, uint size = 10)
{
  // ...
}

kernel void bar(cl::global_ptr<uint[]> input, uint size)
{
  // A kernel function cannot be called by another kernel function.
  foo(input, size);
}

// A kernel function cannot be overloaded.
kernel void bar(cl::global_ptr<float[]> input, uint size)
{
  // ...
}
```

#### Examples, correct

```cpp
template<class T>
void function_template(cl::global_ptr<T[]> input, uint size)
{
  // ...
}

// Specialization for T = float
template<>
void function_template(cl::global_ptr<float[]> input, uint size)
{
  // ...
}

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
([Marker Types section](LINK_TO_OPENCLCXX_SPEC_HTML#marker-types)). 
* `global`, `constant`, `local` storage classes can be passed only by reference or pointer. 
More details in [Explicit address space storage classes](LINK_TO_OPENCLCXX_SPEC_HTML#explicit-address-space-storage-classes)
section.
* Pointers and references must point to one of the following address spaces: global, local
or constant.

#### OpenCL C++ Specification References

* [OpenCL C++ Programming Language: Kernel Functions](LINK_TO_OPENCLCXX_SPEC_HTML#kernel-functions)

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

* [OpenCL C++ Programming Language: Restrictions](LINK_TO_OPENCLCXX_SPEC_HTML#opencl_cxx_restrictions)

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

* [OpenCL C++ Standard Library](LINK_TO_OPENCLCXX_SPEC_HTML#opencl-c-standard-library)

#### Solution

Adding a using-directive `using namespace cl;` right after including all required headers can reduce work needed to port OpenCL C programs to OpenCL C++.

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

### <a name="S-OpenCLCXXSTL-AddressSpacesLibrary"></a>Address Spaces Library

As mentioned in [End of explicit named address spaces](#S-OpenCLCXX-EndOfExplicitNamedAddressSpaces), in OpenCL
C++ explicit named address spaces known from OpenCL C were replaced by explicit address space storage and pointer classes.

**Explicit address space storage classes:**

* `cl::global<T> x` - allocated in global memory.
  * The global storage class can only be used to declare variables at program, function and class scope.
  * The variables at function and class scope must be declared with `static` specifier.
* `cl::local<T> x` - allocated in local memory.
  * The local storage class can only be used to declare variables at program, kernel and class scope.
  * The variables at class scope must be declared with `static` specifier.
* `cl::priv<T> x` - allocated in private memory.
  * The priv storage class cannot be used to declare variables in the program scope, with static specifier or extern specifier.
* `cl::constant<T> x {0}` - allocated in global memory, read-only.
  * The constant storage class can only be used to declare variables at program, kernel and class scope.
  * The variables at class scope must be declared with static specifier.

**Explicit address space storage pointers classes:**

* `cl::global_ptr<T>`
* `cl::local_ptr<T>`
* `cl::private_ptr<T>`
* `cl::constant_ptr<T>`

The explicit address space pointer classes are just like pointers: they can be converted to and from pointers with compatible address spaces, qualifiers and types. Assignment or casting between explicit pointer types of incompatible address spaces is illegal.

All named address spaces are incompatible with all other address spaces, but local, global and private pointers can be converted to standard C++ pointers.

#### Restrictions

[The OpenCL C++ specification](LINK_TO_OPENCLCXX_SPEC_HTML) specification in subsections [3.4.5. Restrictions](LINK_TO_OPENCLCXX_SPEC_HTML#restrictions-2) of section [Address Spaces Library](LINK_TO_OPENCLCXX_SPEC_HTML#address-spaces-library) contains detailed list of restrictions with examples regarding explicit address space storage and pointer classes.
It is very important to read and understand those restrictions.

#### OpenCL C++ Specification References

* [OpenCL C++ Programming Language: Address Spaces](LINK_TO_OPENCLCXX_SPEC_HTML#address-spaces)
* [OpenCL C++ Standard Library: Address Spaces Library](LINK_TO_OPENCLCXX_SPEC_HTML#address-spaces-library)

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
{

}

// Explicit address space storage object passed by pointer
kernel void example_kernel(cl::global<int> * input)
{

}
```

##### Note
> More examples on address spaces can be found in subsections [3.4.5. Restrictions](LINK_TO_OPENCLCXX_SPEC_HTML#restrictions-2) and [3.4.6. Examples](LINK_TO_OPENCLCXX_SPEC_HTML#examples-3) of section [Address Spaces Library](LINK_TO_OPENCLCXX_SPEC_HTML#address-spaces-library) in [OpenCL C++ specification](LINK_TO_OPENCLCXX_SPEC_HTML).

### TODO

* New C++ interfaces for OpenCL C special types
  * pipe is not a keyword anymore, pips is a class now
  * Atomic integer and floating-point types are now classes (but seem to be compatible
  with OpenCL C via typedefs)
* bool and booln types are use in many function instead of int, intn, long, longn
types
* all, any and select functions are slightly different because of using booln type
* (type traits) I think we can mention useful tools for vectors like:
`make_vector<T, N>`, `scalar_type<T>` etc.
* half wrapper?
* OpenCL C `convert_*` built-in functions were replaced by `convert_cast<>` function template.

---
## <a name="S-OpenCLCXXCompilation"></a>OpenCL C++ Compilation Process

### TODO

* Now there are two compilers: front-compiler (OpenCL C++ to SPIR-V) and
back-compiler (SPIR-V to device machine code).
I think it's worth explaining it to the user (with some examples).
* New attributes: max_size, required_num_sub_groups, ivdep


# <a name="S-Bibliography"></a>Bibliography

* [The OpenCL C++ 1.0 Specification](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-cplusplus.pdf)
* [The OpenCL 2.2 API Specification](https://www.khronos.org/registry/OpenCL/specs/opencl-2.2-environment.pdf)
* [The OpenCL C 2.0 Language Specification](https://www.khronos.org/registry/OpenCL/specs/opencl-2.0-openclc.pdf)
* [The OpenCL 2.1 API Specification](https://www.khronos.org/registry/OpenCL/specs/opencl-2.1.pdf)
* Other related specs and presentations
