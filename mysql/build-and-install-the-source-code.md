# Build and install the source code

Building MySQL from the source code enables you to customize build parameters, compiler optimizations, and installation location.

## Pre-requisites

- libreadline
- libaio

## Build and install

First, change default install directory.

```bash
$ cmake -DCMAKE_INSTALL_PREFIX=/path/to/dir
```

Then build and install the source code.
(8: # of cores in your machine)

```bash
$ make -j8 install
```
