# OpenCL C to OpenCL C++ Porting Guidelines

February 8, 2017

Editors:

* [Jakub Szuppe](#)
* [Your name](#)

This document is a set of guidelines for developers who know OpeCL C and plan to port their kernels to OpenCL C++, and therefore they need to know the main differences between those two kernel languages.
The focus is not on highlighting all the differences, but rather on exposing and explaining those that are the most important, and those that may cause hard to detect bugs when porting from OpenCL C to OpenCL C++.

## OpenCL C++ Programming Language

* (Supported Built-in Data Types) No more OpenCL C vector literals
* Restrictions for kernel functions and kernel parameters
* General restrictions (section 3.9)

## OpenCL C++ Standard Library

* cl namespace
* bool and booln types are use in many function instead of int, intn, long, longn types
* all, any and select functions are slightly different because of using booln type
* (type traits) I think we can mention useful tools for vectors like: make_vector<T, N>, scalar_type<T> etc.
* Atomics are now classes, not set of functions (but seem to be compatible with OpenCL C via typedefs)
* pipe is not a keyword anymore, pips is a class now
* half wrapper?

## Compilation

* now there are two compilers: front-compiler (OpenCL C++ to SPIR-V) and back-compiler (SPIR-V to device machine code)
