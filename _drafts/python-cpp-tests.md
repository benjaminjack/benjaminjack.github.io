---
title: Building and testing a hybrid Python/C++ project
author: Benjamin R. Jack
layout: post
excerpt: None yet...
---

- Outline goals

There are several libraries that make interfacing C++ code with Python relatively easy. My favorite is pybind11, which the Rosetta team recently used to rewrite PyRosetta. Pybind11 allows you to write python wrappers for C++ code with minimal boilerplate code. Pybind11 is well-documented and generating a C++-extension module is a breeze. Going from a module to a proper python package, however, takes some more work. In my mind, a python package based on a hybrid Python/C++ code base should include:

- A build system, such as CMake or make
- Unit tests for python code
- Unit tests for C++ code, independent of python wrappers
- A unified build-and-test command that builds all extension modules and runs both python and C++ tests

The aim of this blog post is to describe how to set up a python package that meets all of the above requirements with minimal external tools or libraries.

Here's what we'll use to meet those requirements

- CMake
- `pybind11`
- `unittest` python module
- Catch-lib unit testing library for C++
- setuptools python module

### A simple `pybind11` project

- C++ code
- Python wrappers

### Configuring the build environment
- CMakeLists

### Building with `setuptools`
- Customize setup.py
### Writing python unit tests
- Python tests
### Writing C++ unit tests with `catch`
- C++ tests with catch lib
### Writing a custom test-runner for `setuptools`
- Customize setup.py for test running

- Wrap-up and docs coming next