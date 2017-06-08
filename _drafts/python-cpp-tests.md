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

Here's what we'll use to meet those requirements:

- CMake
- `pybind11`
- `unittest` python module
- Catch-lib unit testing library for C++
- `setuptools` python module

### A simple `pybind11` project

To start, we'll create a simple `pybind11`-based python module. The module will share the same name as our package, `python_cpp_example`. The directory structure for our package should be familiar to those who have written python packages before, with a few exceptions:

```bash
python_cpp_example/
├── CMakeLists.txt  # CMake configuration file
├── LICENSE
├── README.md
├── build  # build directory for C++ executables
├── lib  # external C++ libraries
├── python_cpp_example  # source code (python and C++)
├── setup.py
└── tests  # unit tests (python and C++)
```

Under `lib/`, download and extract the latest `pybind11` release:

```bash
wget https://github.com/pybind/pybind11/archive/v2.1.1.tar.gz
tar -xvf v2.1.1.tar.gz
```
Now we'll write two simple C++ functions and then wrap them in python with `pybind11`. Under `python_cpp_example`, create three files: `math.hpp`, `math.cpp`, and `bindings.cpp`. 

`math.hpp` and `math.cpp` are standard C++ files:

`math.hpp`:
```cpp
/*! Add two integers
    \param i an integer
    \param j another integer
*/
int add(int i, int j);
/*! Subtract one integer from another 
    \param i an integer
    \param j an integer to subtract from \p i
*/
int subtract(int i, int j);
```

`math.cpp`:
```cpp
#include "math.hpp"

int add(int i, int j)
{
    return i + j;
}

int subtract(int i, int j)
{
    return i - j;
}
```
Lastly, we'll define our python wrappers as follows. See the `pybind11` tutorial for more information.

`bindings.cpp`:
```cpp
#include <pybind11-2.1.1/pybind11.h>
#include "math.hpp"

namespace py = pybind11;

PYBIND11_PLUGIN(python_cpp_example)
{
    py::module m("python_cpp_example");
    m.def("add", &add);
    m.def("subtract", &subtract);
    return m.ptr();
}
```
We've now defined two functions in C++, `add` and `subtract`, and written the code to wrap them in a python module called `python_cpp_example`. Your directory structure should now look something like this:

```bash
python_cpp_example/
├── CMakeLists.txt
├── LICENSE
├── README.md
├── build
├── lib
│   └── pybind11-2.1.1
├── python_cpp_example
│   ├── bindings.cpp
│   ├── math.cpp
│   └── math.hpp
├── setup.py
└── tests
``` 

Next we have to set up our build environment. We'll use CMake to build the C++ extension modules in our package.

### Configuring the build environment

We could have written a Makefile directly to build our package, but using CMake vastly simplifies building pybind11-based modules. (If you don't believe me, check out the Makefile that CMake generates.) CMake will also make it easier to add C++ unit tests later.

To start, define a project, set the source directory, and define a list of C++ sources _without_ `bindings.cpp`. (This list will come in handy later when we want build C++ tests independently of any python bindings.) Add the following to your `CMakeLists.txt` file:

```cmake
cmake_minimum_required(VERSION 2.8.12)
project(python_cpp_example)
# Set source directory
set(SOURCE_DIR "python_cpp_example")
# Tell CMake that headers are also in SOURCE_DIR
include_directories(${SOURCE_DIR})
set(SOURCES "${SOURCE_DIR}/math.cpp")
```

Then add the `pybind11` library and define an extension module. Make sure `bindings.cpp` is added to the source list.

```cmake
# Generate python module
add_subdirectory(lib/pybind11-2.1.1)
pybind11_add_module(python_cpp_example ${SOURCES} "${SOURCE_DIR}/bindings.cpp")
```

Next, rather than run CMake directly, we're going to configure python's `setuptools` to build our package automatically via `setup.py`.

### Building with `setuptools`
On its own, `setup.py` will not build an extension module with a CMake-based build system. We have to define a custom build command. The code I'm presenting here was largely taken from the `cmake_example` repository. First, add these two class definitions to `setup.py`.

```python
import os
import re
import sys
import platform
import subprocess

from distutils.version import LooseVersion
from setuptools import setup, Extension
from setuptools.command.build_ext import build_ext


class CMakeExtension(Extension):
    def __init__(self, name, sourcedir=''):
        Extension.__init__(self, name, sources=[])
        self.sourcedir = os.path.abspath(sourcedir)


class CMakeBuild(build_ext):
    def run(self):
        try:
            out = subprocess.check_output(['cmake', '--version'])
        except OSError:
            raise RuntimeError(
                "CMake must be installed to build the following extensions: " +
                ", ".join(e.name for e in self.extensions))

        if platform.system() == "Windows":
            cmake_version = LooseVersion(re.search(r'version\s*([\d.]+)',
                                         out.decode()).group(1))
            if cmake_version < '3.1.0':
                raise RuntimeError("CMake >= 3.1.0 is required on Windows")

        for ext in self.extensions:
            self.build_extension(ext)

    def build_extension(self, ext):
        extdir = os.path.abspath(
            os.path.dirname(self.get_ext_fullpath(ext.name)))
        cmake_args = ['-DCMAKE_LIBRARY_OUTPUT_DIRECTORY=' + extdir,
                      '-DPYTHON_EXECUTABLE=' + sys.executable]

        cfg = 'Debug' if self.debug else 'Release'
        build_args = ['--config', cfg]

        if platform.system() == "Windows":
            cmake_args += ['-DCMAKE_LIBRARY_OUTPUT_DIRECTORY_{}={}'.format(
                cfg.upper(),
                extdir)]
            if sys.maxsize > 2**32:
                cmake_args += ['-A', 'x64']
            build_args += ['--', '/m']
        else:
            cmake_args += ['-DCMAKE_BUILD_TYPE=' + cfg]
            build_args += ['--', '-j2']

        env = os.environ.copy()
        env['CXXFLAGS'] = '{} -DVERSION_INFO=\\"{}\\"'.format(
            env.get('CXXFLAGS', ''),
            self.distribution.get_version())
        if not os.path.exists(self.build_temp):
            os.makedirs(self.build_temp)
        subprocess.check_call(['cmake', ext.sourcedir] + cmake_args,
                              cwd=self.build_temp, env=env)
        subprocess.check_call(['cmake', '--build', '.'] + build_args,
                              cwd=self.build_temp)
        print()  # Add an empty line for cleaner output
```

Next modify `setup()` with the newly-defined custom extension builder:

```python
setup(
    name='python_cpp_example',
    version='0.1',
    author='Benjamin Jack',
    author_email='benjamin.r.jack@gmail.com',
    description='A hybrid Python/C++ test project',
    long_description='',
    # add extension module
    ext_modules=[CMakeExtension('python_cpp_example')],
    # add custom build_ext command
    cmdclass=dict(build_ext=CMakeBuild),
    zip_safe=False,
)
```

Now you should be able to run `python3 setup.py develop` from within your packages root directory and you should see an extension module generated. You can now import and use your new package. 

Up until now, I have largely followed along with the `pybind11` tutorial and CMake example repository. Next I will describe how to add unit testing within this set up.

### Writing python unit tests
- Python tests
### Writing C++ unit tests with `catch`
- C++ tests with catch lib
### Writing a custom test-runner for `setuptools`
- Customize setup.py for test running

- Wrap-up and docs coming next