# [OpenCL C to OpenCL C++ Porting Guidelines](./OpenCLCToOpenCLCppPortingGuidelines.md)

[This document](./OpenCLCToOpenCLCppPortingGuidelines.md) is a set of guidelines for
developers who know OpenCL™ C and plan to port their kernels to OpenCL C++, and therefore
they need to know the main differences between those two kernel languages.

The main focus is on exposing the most important differences between OpenCL C++ and
OpenCL C, and also those which may cause hard-to-detect bugs when porting to OpenCL C++.
Developers who are familiar with OpenCL C and C++ should find OpenCL C++ easy to learn.

## Background

On May 16, 2017, OpenCL 2.2 was released by Khronos Group
([release note](https://www.khronos.org/news/press/khronos-releases-opencl-2.2-with-spir-v-1.2)).
The most important part of new OpenCL version is support for OpenCL C++ kernel language,
which is defined as a static subset of the C++14 standard. OpenCL C++ introduces the
long-awaited features like classes, templates, lambda expressions, function and operator
overloads, and many other constructs which increase parallel programing productivity
through generic programming.

The aim of [the Porting Guidelines](./OpenCLCToOpenCLCppPortingGuidelines.md)
is to help people who are familiar with OpenCL C and C++ to switch to OpenCL C++.
The focus is not on highlighting all the differences between those two kernel languages,
but rather on exposing and explaining those that are the most important, and those that
may cause hard-to-detect bugs when porting from OpenCL C to OpenCL C++.
In the future the guidelines may also provide chapters or sections about new features
introduced in OpenCL 2.2 and OpenCL C++.

## Contributions and LICENSE

Comments, suggestions for improvements, and contributions are most welcome.
More details are found at [CONTRIBUTING](./CONTRIBUTING.md) and [LICENSE](./LICENSE.txt).

## Trademarks

OpenCL and the OpenCL logo are trademarks of Apple Inc. used by permission by Khronos.
Other names are for informational purposes only and may be trademarks of their respective owners.
