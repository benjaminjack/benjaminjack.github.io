---
title: Hybrid Python/C++ packages, revisited
author: Benjamin R. Jack
layout: post
excerpt: Blank
---

Last year I published a blog post that explained how to structure a Python package with a C++ extension module. I laid out four basic requirements for what the package should include:

- An interface for a build system, such as CMake or make
- Unit tests for Python code
- Unit tests for C++ code, independent of Python wrappers
- A unified build-and-test command that builds all extension modules and runs both Python and C++ tests

At the time that I wrote my original post, I had found a satisfactory package structure with `Pybind11`, CMake, Catch, and Python's setuptools.
Well, seven months later, I'm revisiting Python/C++ packaging. After falling down the rabbithole of python packaging, build, and distribution systems, I'm now convinced that the structure that I described in my original post is not ideal. This post will lay out the problems I discovered with my prior approach, and a complete guide to my new approach. I'll conclude with some thoughts on the big picture of working on projects with mixed codebases.

The first part of this post assumes familiarity with my last blog post. If you just want to know how to set up your package, you can skip ahead. If you're even more impatient, you can go right to the complete working example on Github.

### Polluted namespaces and import madness



### A better package structure

#### C++ code and unit tests

#### Python code and unit tests

### Final thoughts: Why test both Python and C++?

In this blog post, I've described how to structure a Python package with a C++ extension module where both the Python bindings and the C++ code are tested independently. I have a large project structured this way, with a set of Python tests and C++ tests. While this structure is less fragile than the previous set up, I'm wondering whether its worthwhile to test both code bases in the same package. Ask yourself two questions,

1. Will my users only interact with my software via the Python interface?
2. Should my C++ code be available as an independent library?

If your answer to question 1 is yes, then just write Python tests. Pybind11 has its own suite of tests, and you can assume that the binding library will work as expected. Python is much quicker to write and easier to maintain, so write your tests in python.

If your C++ code will be available as an independent library, then break out the C++ code into its own github repository. Write C++ tests within that repository. Document, test, and maintain that library indepedently of your Python bindings. For your Python bindings, write a minimal set of tests. Or you could write no tests for the bindings at all. Again, Pybind11 has its own tests, and there are tools that will even autogenerate python bindings from C++ code. Leave it to the pybind11 developers to test their binding library.

So in conclusion, while I've presented a structure here that is more maintainable than before, I'm not convinced that the original goals of this series of posts make sense. Namely, that the Python code and C++ code should be tested independently. If these code bases truly need to be tested indepedently, then they should be developed and maintained independently in separate packages. If you only expect your users to interact with your software via Python, then just test the Python interface.



--------------------------

For my research, I've spent the better part of the last year developing a simulation tool in Python. Python abstracts away things like memory management and type information, making it a great language for working through high-level design decisions. But for simulation software, pure Python is slow. So I've taken to a workflow of prototyping in Python and then rewriting portions of the code base into C++ for performance. I'm left with a high-performance Python package that mixes both Python modules and compiled C++-based extension modules. This combination leverages both the simplicity of Python and the efficiency of C++.

Interfacing C++ code with Python has become relatively easy thanks to several libraries. My favorite is [`pybind11`](http://pybind11.readthedocs.io/en/stable/index.html), a header-only C++ library which takes inspiration from `Boost.Python` but vastly simplifies the syntax. `Pybind11` allows you to write Python wrappers (also called bindings) for C++ code and generate a Python extension module with minimal boilerplate code. Going from a module to a proper Python package, however, takes some more work. In my mind, a Python package based on a hybrid Python/C++ code base should include:

- A build system, such as CMake or make
- Unit tests for Python code
- Unit tests for C++ code, independent of Python wrappers
- A unified build-and-test command that builds all extension modules and runs both Python and C++ tests

The aim of this blog post is to describe how to set up a Python package that meets all of the above requirements with minimal external tools or libraries.

The first part of this post closely follows the [`pybind11` introductory tutorial](http://pybind11.readthedocs.io/en/stable/basics.html) and the [pybind11/CMake example repository](http://pybind11.readthedocs.io/en/stable/compiling.html#building-with-cmake). This post is not meant as an introduction to `pybind11`. If you've never used `pybind11` before, it's worth taking the time now to read through some of the [excellent documentation](http://pybind11.readthedocs.io/en/stable/index.html). For Part 1 of this post, I'll assume a general familiarity with `pybind11`.

In the second part of this post, I'll introduce unit testing frameworks for both the Python side and C++ side of our package's code base. For that, we'll use Python's built-in `unittest` module and the [`catch` C++ library](). Lastly, I'll demonstrate how to stitch these testing frameworks together with Python's `setuptools` and `setup.py`.

All the code presented here has been tested with Python 3.6 and a compiler that supports C++11 on macOS Sierra. C++11 support is required by `pybind11`, but the Python code and extension modules generated here should work with either Python 2 or 3. The code should also work on Windows, although I have not tested it. All code in this blog post is available as a complete working example in this [github repository](https://github.com/benjaminjack/python_cpp_example).

### Part 1: A simple `pybind11` project

#### The C++ code

To start, we'll create a simple `pybind11`-based Python module. The module will share the same name as our package, `python_cpp_example`. The directory structure for our package should be familiar to those who have written Python packages before, with a few exceptions, notably `lib`, `build`, and `CMakeLists.txt`:

```bash
python_cpp_example/
├── build  # build directory for C++ executables
├── lib  # external C++ libraries
├── python_cpp_example  # source code (Python and C++)
├── setup.py
└── tests  # unit tests (Python and C++)
```

The `build/` directory will contain compiled code generated by our build system. The `lib/` directory will contain the C++ libraries needed for our package. Under `lib/`, download and extract the latest `pybind11` release by running the following commands (assuming you're working in a *nix environment):

```bash
wget https://github.com/pybind/pybind11/archive/v2.1.1.tar.gz
tar -xvf v2.1.1.tar.gz
# Copy pybind11 library into our project
cp -r pybind11-2.1.1 python_cpp_example/lib/pybind11
```

Now we'll write two simple C++ functions and then wrap them in Python with `pybind11`. Under `python_cpp_example`, create three files: `math.hpp`, `math.cpp`, and `bindings.cpp`. 

`math.hpp` and `math.cpp` are simple C++ header and definition files:

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

Lastly, we'll define our Python wrappers in a file called `bindings.cpp`. See the [`pybind11` tutorial](http://pybind11.readthedocs.io/en/stable/basics.html) for a detailed explanation of what each line of code does here.

`bindings.cpp`:
```cpp
#include <pybind11/pybind11.h>
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
We've now defined two functions in C++, `add` and `subtract`, and written the code to wrap them in a Python module called `python_cpp_example`. Your directory structure should now look something like this:

```bash
python_cpp_example/
├── build
├── lib
│   └── pybind11
├── python_cpp_example
│   ├── bindings.cpp
│   ├── math.cpp
│   └── math.hpp
├── setup.py
└── tests
``` 

Next we have to set up our build environment. We'll use CMake to build the C++ extension modules in our package.

#### Configuring the build environment

We could have written a Makefile directly to build our package, but using CMake simplifies building `pybind11`-based modules. (If you don't believe me, check out the Makefile that CMake generates.) Using CMake will also make it easier to add C++ unit tests later.

First make sure that you have [CMake](https://cmake.org/) installed. On macOS, installation can be done with [`brew`](https://brew.sh) by running the following command:

```bash
> brew install cmake
```

If you are not familiar with CMake, I suggest skimming through the [CMake introductory tutorial](https://cmake.org/cmake-tutorial/). We're going to use a small set of CMake functions here, so even if you're new to CMake, the code should be easy to follow.

To start, we'll define a project, set the source directory, and define a list of C++ sources _without_ `bindings.cpp`. (This list will come in handy later when we want build C++ tests independently of any Python bindings.) Create a `CMakeLists.txt` file in the package's root directory and add the following:

```cmake
cmake_minimum_required(VERSION 2.8.12)
project(python_cpp_example)
# Set source directory
set(SOURCE_DIR "python_cpp_example")
# Tell CMake that headers are also in SOURCE_DIR
include_directories(${SOURCE_DIR})
set(SOURCES "${SOURCE_DIR}/math.cpp")
```

Next, we'll tell CMake to add the `pybind11` directory to our project and define an extension module. This time, make sure `bindings.cpp` is added to the sources list. Add the following to `CMakeLists.txt`:

```cmake
# Generate Python module
add_subdirectory(lib/pybind11)
pybind11_add_module(python_cpp_example ${SOURCES} "${SOURCE_DIR}/bindings.cpp")
```

That's all we need to instruct CMake to build our extension module. Rather than run CMake directly, however, we're going to configure Python's built-in [`setuptools`](http://setuptools.readthedocs.io/en/latest/) to build our package automatically via `setup.py`.

#### Building with `setuptools`

On its own, `setup.py` will not build an extension module with a CMake-based build system. We have to define a custom build command. The code I'm presenting here was largely taken from the [`pybind11`'s CMake example repository](https://github.com/pybind/cmake_example). I won't explain every line of the code here, but in brief, we're defining two classes that will create a temporary build directory and then call CMake to build any extension modules in our package. Add these two class definitions to `setup.py`:

```python
import os
import re
import sys
import sysconfig
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

Next, at the bottom of `setup.py`, modify `setup()` with the newly-defined custom extension builder:

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

Now you should be able to run `python3 setup.py develop` from within your package's root directory and you will see an extension module generated. You can now import and use your new package. 

Up until now, I have largely followed along with the [`pybind11` tutorial](http://pybind11.readthedocs.io/en/stable/basics.html) and [`pybind11`'s CMake example repository](https://github.com/pybind/cmake_example). In Part 2, I will describe how to add unit testing within this set up.

### Part 2: Adding unit tests

#### Writing Python unit tests

We'll begin by adding Python unit tests using Python's built-in `unittest` module. Under `tests/` add an empty file called `__init__.py`. This file will stay empty, but it is required for `unittest`'s automatic test discovery.

```bash
python_cpp_example/
├── CMakeLists.txt
├── build
├── lib
│   └── pybind11
├── python_cpp_example
│   ├── bindings.cpp
│   ├── math.cpp
│   └── math.hpp
├── setup.py
└── tests
    └── __init__.py
```

In the same `tests/` directory, add a file `math_test.py` with a few simple unit tests.

```python
import unittest
import python_cpp_example  # our `pybind11`-based extension module

class MainTest(unittest.TestCase):
    def test_add(self):
        # test that 1 + 1 = 2
        self.assertEqual(python_cpp_example.add(1, 1), 2)

    def test_subtract(self):
        # test that 1 - 1 = 0
        self.assertEqual(python_cpp_example.subtract(1, 1), 0)

if __name__ == '__main__':
    unittest.main()
```

That's all you need for Python unit tests. You can add as many test files as you want, and each one should define a class that extends `unittest.TestCase`. As long as all of your files have a `_test.py` suffix, `unittest` will automatically discover them. Run `python3 setup.py test` and you should get output that looks like this:

```bash
> python3 setup.py test
test_add (tests.math_test.MainTest) ... ok
test_subtract (tests.math_test.MainTest) ... ok

----------------------------------------------------------------------
Ran 2 tests in 0.001s

OK
```

Python's built-in `unittest` module is a powerful unit testing framework with a variety of built in assertions. You can read more about `unittest` in the [official Python documentation](https://docs.python.org/3/library/unittest.html).

#### Writing C++ unit tests with `catch`

Unlike Python, C++ needs an external library to enable unit testing. I've chosen to use [`catch`](http://catch-lib.net) for its concise syntax and its header-only structure. Download and extract `catch` in the `lib/` directory of your package. On a *nix system, you could run the following:

```bash
cd lib
wget https://github.com/philsquared/Catch/archive/v1.9.4.tar.gz
tar -xvf v1.9.4.tar.gz
# Copy catch library into our project
cp -r catch-1.9.4 python_cpp_example/lib/catch 
```

Similar to Python's `__init__.py`, `catch` requires an initialization file that we'll name `test_main.cpp`. This configuration file must contain two specific lines and nothing else. Under the `tests/` directory, add the following file:

`test_main.cpp`:
```cpp
#define CATCH_CONFIG_MAIN
#include <catch.hpp>
```

Now we'll make a file with two simple unit tests. This time I'm using a `test_` prefix (rather than a `_test.py` suffix) to easily distinguish between Python unit tests and C++ unit tests without looking at the file extension. Add the following to a file named `test_math.cpp` in the `test/` directory:

`test_math.cpp`
```cpp
#include <catch.hpp>

#include "math.hpp"

TEST_CASE("Addition and subtraction")
{
    REQUIRE(add(1, 1) == 2);
    REQUIRE(subtract(1, 1) == 0);
}
```

These tests are analogous to the Python tests in the previous section. Normally, I would not unit test both the `pybind11` Python wrappers and the underlying C++ definitions for such simple functions. However, you can imagine an instance in which you didn't want to expose all of your C++ code with Python wrappers, but you still wanted to unit test that C++ code. Likewise, the Python wrappers can get quite complex and it may be useful to test your C++ code independently of the wrapping code.

Your directory structure should now look something like this:

```bash
python_cpp_example/
├── CMakeLists.txt
├── LICENSE
├── README.md
├── build
│   └── temp.macosx-10.12-x86_64-3.6
├── lib
│   ├── catch
│   └── pybind11
├── python_cpp_example
│   ├── bindings.cpp
│   ├── math.cpp
│   └── math.hpp
├── python_cpp_example.cpython-36m-darwin.so
├── setup.py
└── tests
    ├── __init__.py
    ├── math_test.py
    ├── test_main.cpp
    └── test_math.cpp
```

Lastly, we need to instruct CMake that we've added C++ unit tests. We'll add `test_main.cpp` and `test_math.cpp` to a `TESTS` variable. Then we'll include the `catch` library and define an executable `python_cpp_example_test`. Add the following to your `CMakeLists.txt` file:

```cmake
SET(TEST_DIR "tests")
SET(TESTS ${SOURCES}
    "${TEST_DIR}/test_main.cpp"
    "${TEST_DIR}/test_math.cpp")

# Generate a test executable
include_directories(lib/catch/include)
add_executable("${PROJECT_NAME}_test" ${TESTS})
```

Now run `python3 ./setup.py develop` and if you navigate to `build/temp.*` (e.g., `build/temp.macosx-10.12-x86_64-3.6` on my system), you should see an executable `python_cpp_example_test`. Running this executable will execute the `catch` unit tests. Rather than run this executable ourselves, however, we will tell `setuptools` to run it along side the Python unit tests.

#### Writing a custom test-runner for `setuptools`

Just like we wrote a custom extension builder in `setup.py` in Part 1, we'll write a custom test runner to run both Python and C++ unit tests. Add the following import command and class definition to your `setup.py` file: 

```python
from setuptools.command.test import test as TestCommand

class CatchTestCommand(TestCommand):
    """
    A custom test runner to execute both Python unittest tests and C++ Catch-
    lib tests.
    """
    def distutils_dir_name(self, dname):
        """Returns the name of a distutils build directory"""
        dir_name = "{dirname}.{platform}-{version[0]}.{version[1]}"
        return dir_name.format(dirname=dname,
                               platform=sysconfig.get_platform(),
                               version=sys.version_info)

    def run(self):
        # Run Python tests
        super(CatchTestCommand, self).run()
        print("\nPython tests complete, now running C++ tests...\n")
        # Run catch tests
        subprocess.call(['./*_test'],
                        cwd=os.path.join('build',
                                         self.distutils_dir_name('temp')),
                        shell=True)
```

We're defining a class with two functions. The first function `distutils_dir_name` is just a helper function that generates a path to the CMake build directory. The second function `run`, first runs the Python unit tests, then runs the C++ unit tests by executing `python_cpp_example_test`. This function will run any executable that it finds in the build directory with a `_test` suffix.

Next define a new `test` command at the bottom of `setup.py`.

`setup.py`:
```python
setup(
    name='python_cpp_example',
    version='0.1',
    author='Benjamin Jack',
    author_email='benjamin.r.jack@gmail.com',
    description='A hybrid Python/C++ test project',
    long_description='',
    ext_modules=[CMakeExtension('python_cpp_example')],
    # add custom test command
    cmdclass=dict(build_ext=CMakeBuild, test=CatchTestCommand),
    zip_safe=False,
)
```

And now, the moment we've been waiting for: a single command to build and test a hybrid Python/C++ package.

```bash
> python3 setup.py test
...

test_add (tests.math_test.MainTest) ... ok
test_subtract (tests.math_test.MainTest) ... ok

----------------------------------------------------------------------
Ran 2 tests in 0.001s

OK

Python tests complete, now running C++ tests...

===============================================================================
All tests passed (2 assertions in 1 test case)
```

Under the hood, the above command is first building any extension modules, executing Python unit tests, and then executing C++ unit tests.

### Wrap-up

You should now have a functioning, minimal Python/C++ package that is set up to unit test both the Python and C++ code. For a complete working example, see the [github repository](https://github.com/benjaminjack/python_cpp_example) that accompanies this blog post. In my next blog post, we'll learn how to auto-generate documentation for the mixed-language code base presented here.
