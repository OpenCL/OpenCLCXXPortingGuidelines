# OpenCL C to OpenCL C++ Porting Guidelines

February 8, 2017

Editors:

* [Jakub Szuppe](#)
* [Your name](#)

This document is a set of guidelines for developers who know OpeCL C and plan to port their kernels to OpenCL C++,
and therefore they need to know the main differences between those two kernel languages.
The focus is not on highlighting all the differences, but rather on exposing and explaining those that are the most
important, and those that may cause hard to detect bugs when porting from OpenCL C to OpenCL C++.

---
## OpenCL C++ Programming Language

### OpenCL C vector literals

TODO

### Restrictions for kernel functions and kernel parameters

TODO

### General restrictions

TODO

---
## OpenCL C++ Standard Library

OpenCL C++ does not support the C++14 standard library, but instead implements its own standard library. It is
a replacement for built-in functions provided in OpenCL C.

##### Note
> OpenCL C++ classes and functions are not auto-included.

### Namespace cl::

All class and funtions provided in OpenCL C++ Standard Library are located in namespace `cl::`.

#### Example

A using-directive `using namespace cl;` can be used to reduce work need to port OpenCL C programs
to OpenCL C++. However, it is important to remember to include required headers.

```cpp
#include <opencl_memory>
#include <opencl_integer> // cl::abs(gentype x)

kernel void foo(cl::global_ptr<int[]> input, uint size)
{
  uint global_id = cl::get_global_id(0); // note cl:: prefix
  if(global_id < size)
  {
    using namespace cl;
    input[global_id] = abs(input[global_id]); // now no need for cl:: here
  }
}
```

### TODO

* bool and booln types are use in many function instead of int, intn, long, longn types
* all, any and select functions are slightly different because of using booln type
* (type traits) I think we can mention useful tools for vectors like: make_vector<T, N>, scalar_type<T> etc.
* Atomics are now classes, not set of functions (but seem to be compatible with OpenCL C via typedefs)
* pipe is not a keyword anymore, pips is a class now
* half wrapper?

---
## Compilation

TODO

* now there are two compilers: front-compiler (OpenCL C++ to SPIR-V) and back-compiler (SPIR-V to device machine code)
