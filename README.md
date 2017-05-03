# [OpenCL C to OpenCL C++ Porting Guidelines](./OpenCLCToOpenCLCppPortingGuidelines.md)

On May 16, 2017, OpenCL 2.2 was released by Khronos Group ([release note](#)).
The most important part of new OpenCL version is support for OpenCL C++ kernel language,
which is defined as a static subset of the C++14 standard. OpenCL C++ introduces the
long-awaited features like classes, templates, lambda expressions, function and operator
overloads, and many other constructs which increase parallel programing productivity
through generic programming.

This document is a set of guidelines for developers who know OpenCL C and plan to
port their kernels to OpenCL C++, and therefore they need to know the main
differences between those two kernel languages.
The focus is not on highlighting all the differences, but rather on exposing
and explaining those that are the most important, and those that may cause
hard-to-detect bugs when porting from OpenCL C to OpenCL C++.

Comments, suggestions for improvements, and contributions are most welcome.

## Contributions and LICENSE

Comments and suggestions for improvements are most welcome. More details are found at [CONTRIBUTING](./CONTRIBUTING.md) and [LICENSE](./LICENSE.txt).
