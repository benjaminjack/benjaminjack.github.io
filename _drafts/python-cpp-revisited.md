---
title: Hybrid Python/C++ packages, revisited
author: Benjamin R. Jack
layout: post
excerpt: Last year I published a blog post that explained how to structure a Python package with a C++ extension module. My goal was to craft a Python package that leveraged C++ for performance and had an easily maintainable and testable structure. Well, seven months later, I'm revisiting Python/C++ packaging. I'm now convinced that the structure that I described in my original post is not ideal. This post will lay out the problems I discovered with my prior approach, and a complete guide to my new approach. I'll conclude with some thoughts on the big picture of working on projects with mixed codebases.
---

Last year I published a blog post that explained [how to structure a Python package with a C++ extension module](2017/06/12/python-cpp-tests.html). My goal was to craft a Python package that leveraged C++ for performance and had an easily maintainable and testable structure. Well, seven months later, I'm revisiting Python/C++ packaging. I'm now convinced that the structure that I described in my original post is not ideal. This post will lay out the problems I discovered with my prior approach, and a complete guide to my new approach. I'll conclude with some thoughts on the big picture of working on projects with mixed codebases.

The first part of this post assumes familiarity with [my last blog post](2017/06/12/python-cpp-tests.html). If you just want to know how to set up your package, you can skip ahead. If you're even more impatient, you can go right to the [complete working example on Github](https://github.com/benjaminjack/python_cpp_example).

### Part 1: Polluted namespaces and import madness

Python's package import and namespace system is complex, to say the least. [There are many pitfalls](http://python-notes.curiousefficiency.org/en/latest/python_concepts/import_traps.html). Recall the repository directory structure that I described in my previous post:

```bash
python_cpp_example
├── CMakeLists.txt
├── LICENSE
├── README.md
├── build/
├── lib
│   ├── catch/
│   └── pybind11/
├── python_cpp_example
│   ├── __init__.py
│   ├── bindings.cpp
│   ├── math.cpp
│   └── math.hpp
├── setup.py
└── tests
    ├── __init__.py
    ├── math_test.py
    ├── test_main.cpp
    └── test_math.cpp
```

This directory structure is [common](http://docs.python-guide.org/en/latest/writing/structure/). The root directory `python_cpp_example/` contains a LICENSE, README.md, setup.py and other files that support the `python_cpp_example` package. The package source code lives in a subdirectory also named `python_cpp_example/`. Strictly speaking, a python package is a collection of a python modules in a directory with an `__init__.py` file that tells python, "Hey, this is a package." So in this example, the `python_cpp_example/` *subdirectory* is really the package. 

When actively developing a Python package, developers commonly install packages in development mode so they can make changes to the source code without having to reinstall the package after every change. In our example, running `setup.py develop` will add the `python_cpp_example` package to the global python import namespace. This behavior is desirable so we can import our installed package from anywhere by running `import python_cpp_example`. However, without specifying in `setup.py` exactly where our package source code lives, running `setup.py develop` will add *any* subdirectory with an `__init__.py` to the global namespace. Notice a problem with the directory structure above? The `tests` directory also has an `__init__.py`, making it a package that `setuptools` will install. Running `setup.py develop` then makes this possible:

```python
import tests  # Uh-oh
tests.test_math.test_add()
```

We've polluted the global package namespace with a generic-sounding `tests` package. This is definitely not what we want. So what can we do? One option would be to never install our package development mode, but this is a fragile solution. Our package should behave the same whether it is installed into `site-packages/` or in development mode. The safer option is to change the directory structure and specify source directories in `setup.py` to proactively avoid import problems.

### Part 2: A better package structure

Many others have written about [how to structure a Python package](https://www.google.com/search?q=python+package+structure). The most [thorough and well-reasoned post](https://blog.ionelmc.ro/2014/05/25/python-packaging/) I've read suggests the following layout:

```bash
python_cpp_example
├── setup.py
├── src
│   └── python_cpp_example/  # Source code goes under this directory
└── tests/  # Tests live here
```

This layout guards against some of the import problems I've described above. To make sure nothing outside of `src/` gets added to the global package namespace, add these two arguments to `setup()` in your `setup.py`:

```python
from setuptools import find_packages

setup(
    ...
    packages=find_packages('src'),
    package_dir={'':'src'},
    ...
)
```

This guarantees that `tests` will never be accidentally added to the global package namespace. Using this layout, we will construct our hybrid Python/C++ package. The following sections follow closely with my [prior blog post](2017/06/12/python-cpp-tests.html), but adapted to this new directory structure.

#### A simple package with a `pybind11`-based C++ extension module

To start, we'll create a simple `pybind11`-based Python module. [`Pybind11`](https://github.com/pybind/pybind11) is an excellent header-only C++ library that makes it easy to write Python wrappers for any C++ code and bundle them into an extension module. The extension module will share the same name as our package, `python_cpp_example`. The directory structure for our package is similar to the one described above, with a few additions, notably `lib`, `build`, and `CMakeLists.txt`:

```bash
python_cpp_example
├── build/  # Build directory for C++ extension modules
├── lib  # External C++ libraries
├── setup.py
├── src
│   └── python_cpp_example/  # Python and C++ source code
└── tests/  # Python and C++ unit tests
```

We'll also assume that our package already contains one module called `hello.py`.

`hello.py`:
```python
def say_hello():
    print("Hello world!")
```

Make sure that the `src/python_cpp_example` directory contains a blank `__init__.py` so Python knows that this directory is a package. We will place our Python and C++ source code in the same directory.

The `build/` directory will contain compiled code generated by our build system. The `lib/` directory will contain the C++ libraries needed for our package. Under `lib/`, download and extract the latest `pybind11` release by running the following commands (assuming you're working in a *nix environment):

```bash
wget https://github.com/pybind/pybind11/archive/v2.1.1.tar.gz
tar -xvf v2.1.1.tar.gz
# Copy pybind11 library into our project
cp -r pybind11-2.1.1 python_cpp_example/lib/pybind11
```

Now we'll write two simple C++ functions and then wrap them in Python with `pybind11`. Under `src/python_cpp_example`, create three files: `math.hpp`, `math.cpp`, and `bindings.cpp`. 

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
python_cpp_example
├── build
├── lib
│   └── pybind11
├── src
│   └── python_cpp_example
│       ├── __init__.py
│       ├── hello.py
│       ├── bindings.cpp
│       ├── math.cpp
│       ├── math.hpp
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
set(SOURCE_DIR "src/python_cpp_example")
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
from setuptools import setup, find_packages, Extension
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
    # tell setuptools to look for any packages under 'src'
    packages=find_packages('src'),
    # tell setuptools that all packages will be under the 'src' directory
    # and nowhere else
    package_dir={'':'src'},
    # add an extension module named 'python_cpp_example' to the package 
    # 'python_cpp_example'
    ext_modules=[CMakeExtension('python_cpp_example/python_cpp_example')],
    # add custom build_ext command
    cmdclass=dict(build_ext=CMakeBuild),
    zip_safe=False,
)
```

Now you should be able to run `python3 setup.py develop` from within your package's root directory and you will see an extension module generated under `src/python_cpp_example/`. You can now import and use your new package. 

Up until now, I have largely followed along with the [`pybind11` tutorial](http://pybind11.readthedocs.io/en/stable/basics.html) and [`pybind11`'s CMake example repository](https://github.com/pybind/cmake_example). In Part 2, I will describe how to add unit testing within this set up.

#### Writing Python unit tests

We'll begin by adding Python unit tests using Python's built-in `unittest` module. Under `tests/` add an empty file called `__init__.py`. This file will stay empty, but it is required for `unittest`'s automatic test discovery.

```bash
python_cpp_example/
├── CMakeLists.txt
├── build/
├── lib
│   ├── catch/
│   └── pybind11/
├── setup.py
├── src
│   └── python_cpp_example
│       ├── __init__.py
│       ├── hello.py
│       ├── bindings.cpp
│       ├── math.cpp
│       ├── math.hpp
│       └── python_cpp_example.cpython-36m-darwin.so
└── tests
    └── __init__.py
```

In the same `tests/` directory, add a file `math_test.py` with a few simple unit tests.

```python
import unittest
# import our `pybind11`-based extension module from package python_cpp_example
from python_cpp_example import python_cpp_example

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

Add the following `test_suite` keyword argument to `setup()` in your `setup.py` script:

```python
setup(
    ...
    test_suite='tests',
    ...
)
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

These tests are analogous to the Python tests in the previous section. Normally, I would not unit test both the `pybind11` Python wrappers and the underlying C++ definitions for such simple functions. However, you can imagine an instance in which you didn't want to expose all of your C++ code with Python wrappers, but you still wanted to unit test that C++ code. Likewise, the Python wrappers can get quite complex and it may be useful to test your C++ code independently of the wrapping code. I'll revisit the topic of unit testing at the end of this post.

Your directory structure should now look something like this:

```bash
python_cpp_example/
├── CMakeLists.txt
├── build
│   └── temp.macosx-10.12-x86_64-3.6
├── lib
│   ├── catch/
│   └── pybind11/
├── src
│   └── python_cpp_example
│       ├── __init__.py
│       ├── hello.py
│       ├── bindings.cpp
│       ├── math.cpp
│       ├── math.hpp
│       └── python_cpp_example.cpython-36m-darwin.so
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

Now run `python3 ./setup.py develop` and if you navigate to `build/temp.*` (e.g., `build/temp.macosx-10.12-x86_64-3.6` on my system), you should see an executable `python_cpp_example_test`. Running this executable will execute the `catch` unit tests. Rather than run this executable ourselves, however, we will move the executable to somewhere more convenient and use `setuptools` to execute it along side our Python unit tests.

#### Using Python's `unittest` to execute C++ `catch` tests

To run our C++ unit tests along side the Python tests, we're going to call the `python_cpp_example_test` executable from a Python unit test. This requires two steps: first moving `python_cpp_example_test` from the build directory to the `tests/` directory, then writing a simple `unittest` test that calls the executable.

First, add this `copy_test_file()` function to the `CMakeBuild` class in `setup.py`:

```python
...
class CMakeBuild(build_ext):
    def run(self):
        ...

    def build_extension(self, ext):
        ...

    def copy_test_file(self, src_file):
        '''
        Copy ``src_file`` to `tests/bin` directory, ensuring parent directory 
        exists. Messages like `creating directory /path/to/package` and
        `copying directory /src/path/to/package -> path/to/package` are 
        displayed on standard output. Adapted from scikit-build.
        '''
        # Create directory if needed
        dest_dir = os.path.join(os.path.dirname(
            os.path.abspath(__file__)), 'tests', 'bin')
        if dest_dir != "" and not os.path.exists(dest_dir):
            print("creating directory {}".format(dest_dir))
            os.makedirs(dest_dir)

        # Copy file
        dest_file = os.path.join(dest_dir, os.path.basename(src_file))
        print("copying {} -> {}".format(src_file, dest_file))
        copyfile(src_file, dest_file)
        copymode(src_file, dest_file)

... 
```

Then call `copy_test_file` from within `CMakeBuild.build_extension`.

`setup.py`:
```python
...
class CMakeBuild(build_ext):
    def run(self):
        ...

    def build_extension(self, ext):
        ... # There is more code here not shown
        subprocess.check_call(['cmake', ext.sourcedir] + cmake_args,
                              cwd=self.build_temp, env=env)
        subprocess.check_call(['cmake', '--build', '.'] + build_args,
                              cwd=self.build_temp)
        # Copy *_test file to tests directory
        test_bin = os.path.join(self.build_temp, 'python_cpp_example_test')
        self.copy_test_file(test_bin)
        print() # Add empty line for nicer output

    def copy_test_file(self, src_file):
        ...
... 
```

Upon building the extension module, Python's setuptools will also now copy `python_cpp_example_test` into the `tests/bin` directory. Next we add another Python unit test under the `tests` directory.

`test_cpp.py`:
```python
import unittest
import subprocess
import os

class MainTest(unittest.TestCase):
    def test_cpp(self):
        print("\n\nTesting C++ code...")
        subprocess.check_call(os.path.join(os.path.dirname(
            os.path.relpath(__file__)), 'bin', 'python_cpp_example_test'))
        print()  # for prettier output


if __name__ == '__main__':
    unittest.main()
```

And now, the moment we've been waiting for: a single command to build and test a hybrid Python/C++ package.

```bash
> python3 setup.py test
...

test_cpp (tests.cpp_test.MainTest) ... 

Testing C++ code...
===============================================================================
All tests passed (2 assertions in 1 test case)


Resuming Python tests...

ok
test_add (tests.math_test.MainTest) ... ok
test_subtract (tests.math_test.MainTest) ... ok

----------------------------------------------------------------------
Ran 3 tests in 0.006s

OK
```

Under the hood, the above command is first building any extension modules, executing Python unit tests, and then executing C++ unit tests.

### Final thoughts: why test both Python and C++?

In this blog post, I've described how to structure a Python package with a C++ extension module where both the Python bindings and the C++ code are tested independently. I have a large project structured this way, with a set of Python tests and a set C++ tests. While the structure described in this post is less fragile than the previous set up, I'm wondering whether it's worthwhile to test both codebases in the same repository. I think it comes down to asking two questions,

1. Will my users only interact with my software via the Python interface?
2. Should my C++ code be available as an independent library?

If your answer to the first question is yes, then just write Python tests. `Pybind11` has its own suite of tests, so trust that the binding library will work as expected. Python is much quicker to write and easier to maintain. 

If your C++ code will be available as an independent library, then break out the C++ code into its own Github repository. Write C++ tests within that repository. Document, test, and maintain that library independently of your Python bindings. For your Python bindings, write a minimal set of tests. Or you could write no tests for the bindings at all. Again, `pybind11` has its own tests, and there are tools that will even [autogenerate python bindings from C++ code](https://github.com/RosettaCommons/binder). Leave it to the `pybind11` developers to test their binding library.

So in conclusion, while I've presented a structure here that is more maintainable than the structure described in my previous post, I'm not convinced that the original motivation for this series of posts make sense. Namely, I question that Python code and C++ code should be tested independently. If these codebases truly need to be tested independently, then they should be developed and maintained independently in separate repositories. Conversely, if you only expect your users to interact with your software via Python, then just test the Python interface. More tests are not always better.