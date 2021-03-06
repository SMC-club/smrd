# Linux Build Instructions

This document focuses on building smrd for development purposes under recent
Ubuntu linux distributions.

Development is regularly done on Ubuntu 16.04 or later. For non Ubuntu
distributions, the steps below should work be installing the appropriate
dependencies using that distribution's package management tools.

## Dependencies

Use `apt-get` to install the dependencies provided by the distribution

```
$ apt-get update
$ apt-get install -y gcc g++ wget git cmake protobuf-compiler libprotobuf-dev libssl-dev
```

Advanced users can choose to install newer versions of gcc, or the clang compiler.
At this time, smrd only supports protobuf version 2. Using version 3 of 
protobuf will give errors.

### Build Boost

Boost 1.67 or later is required. We recommend downloading and compiling boost
with the following process: After changing to the directory where
you wish to download and compile boost, run

``` 
$ wget https://dl.bintray.com/boostorg/release/1.68.0/source/boost_1_68_0.tar.gz
$ tar -xzf boost_1_68_0.tar.gz
$ cd boost_1_68_0
$ ./bootstrap.sh
$ ./b2 headers
$ ./b2 -j<Num Parallel>
```

### (Optional) Dependencies for Building Source Documentation

Source code documentation is not required for running/debugging smrd. That
said, the documentation contains some helpful information about specific
components of the application. For more information on how to install and run
the necessary components, see [this document](../../docs/README.md)

## Build

### Clone the smrd repository

From a shell:

```
git clone git@github.com:CryptoManiac/smrd.git
cd rippled
```

### Configure Library Paths

If you didn't persistently set the `BOOST_ROOT` environment variable to the
directory in which you compiled boost, then you should set it temporarily.

For example, you built Boost in your home directory `~/boost_1_68_0`, you
would do for any shell in which you want to build:

```
export BOOST_ROOT=~/boost_1_68_0
```

Alternatively, you can add `DBOOST_ROOT=~/boost_1_68_0` to the command line when
invoking `cmake`.

### Generate and Build

All builds should be done in a separate directory from the source tree root 
(a subdirectory is fine). For example, from the root of the ripple source tree:

```
mkdir my_build
cd my_build
```

followed by:

```
cmake -DCMAKE_BUILD_TYPE=Debug ..
```

`CMAKE_BUILD_TYPE` can be changed as desired for `Debug` vs.
`Release` builds (all four standard cmake build types are supported).

To select a different compiler (most likely gcc will be found by default), pass 
`-DCMAKE_C_COMPILER=<path/to/c-compiler>` and
`-DCMAKE_CXX_COMPILER=</path/to/cxx-compiler>` when configuring. If you prefer, 
you can instead set `CC` and `CXX` environment variables which cmake will honor.

Once you have generated the build system, you can run the build via cmake:

```
cmake --build . -- -j <parallel jobs>
```

the `-j` parameter in this example tells the build tool to compile several
files in parallel. This value should be chosen roughly based on the number of
cores you have available and/or want to use for building.

When the build completes succesfully, you will have a `smrd` executable in
the current directory, which can be used to connect to the network (when
properly configured) or to run unit tests.

#### Options During Configuration:

The CMake file defines a number of configure-time options which can be
examined by running `cmake-gui` or `ccmake` to generated the build. In
particular, the `unity` option allows you to select between the unity and
non-unity builds. `unity` builds are faster to compile since they combine
multiple sources into a single compiliation unit - this is the default if you
don't specify. `nounity` builds can be helpful for detecting include omissions
or for finding other build-related issues, but aren't generally needed for
testing and running.

* `-Dunity=ON` to enable/disable unity builds (defaults to ON)  
* `-Dassert=ON` to enable asserts
* `-Djemalloc=ON` to enable jemalloc support for heap checking
* `-Dsan=thread` to enable the thread sanitizer with clang
* `-Dsan=address` to enable the address sanitizer with clang
* `-Dstatic=ON` to enable static linking library dependencies

Several other infrequently used options are available - run `ccmake` or
`cmake-gui` for a list of all options.

#### Optional Installation

The smrd cmake build supports an installation target that will install
smrd as well as a support library that can be used to sign transactions. In
order to build and install the files, specify the `install` target when
building, e.g.:

```
cmake -DCMAKE_BUILD_TYPE=Debug -DCMAKE_INSTALL_PREFIX=/opt/local ..
cmake --build . --target install -- -j <parallel jobs>
```

We recommend specifying `CMAKE_INSTALL_PREFIX` when configuring in order to
explicitly control the install location for your files. Without this setting,
cmake will typically install in `/usr/local`. It is also possible to "rehome"
the installation by specifying the `DESTDIR` env variable during the install phase,
e.g.:

```
DESTDIR=~/mylibs cmake --build . --target install -- -j <parallel jobs>
```

in which case, the files would be installed in the `CMAKE_INSTALL_PREFIX` within
the specified `DESTDIR` path.

#### Signing Library

If you want to use the signing support library to create an application, there
are two simple mechanisms with cmake + git that facilitate this.

With either option below, you will have access to a library from the
smrd project that you can link to in your own project's CMakeLists.txt, e.g.:

```
target_link_libraries (my-signing-app Ripple::xrpl_core)
```

##### Option 1: git submodules + add_subdirectory

First, add the smrd repo as a submodule to your project repo:

```
git submodule add -b master https://github.com/CryptoManiac/smrd.git vendor/smrd
```

change the `vendor/smrd` path as desired for your repo layout. Furthermore,
change the branch name if you want to track a different smrd branch, such
as `develop`.
   
Second, to bring this submodule into your project, just add the smrd subdirectory:

```
add_subdirectory (vendor/smrd)
```
    
##### Option 2: installed smrd + find_package

First, follow the "Optional Installation" instructions above to
build and install the desired version of smrd.

To make use of the installed files, add the following to your CMakeLists.txt file:

```
set (CMAKE_MODULE_PATH /opt/local/lib/cmake/ripple ${CMAKE_MODULE_PATH})
find_package(Ripple REQUIRED)
```

change the `/opt/local` module path above to match your chosen installation prefix.
