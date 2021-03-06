# PolyTracker

<p align="center">
  <img src="logo/polytracker_name.png?raw=true" width="256" title="PolyTracker">
</p>
<br />

PolyTracker is a tool for the _Automated Lexical Annotation and Navigation of Parsers_, a backronym devised solely for the purpose of referring to it as _The ALAN Parsers Project_. It is a an LLVM pass that instruments the programs it compiles to track which bytes of an input file are operated on by which functions. It outputs a JSON file containing the function-to-input-bytes mapping. Unlike dynamic instrumentation alternatives like [Taintgrind](https://github.com/wmkhoo/taintgrind), PolyTracker imposes negligible performance overhead for almost all inputs, and is capable of tracking every byte of input at once.

PolyTracker can be used in conjunction with [PolyFile](https://github.com/trailofbits/polyfile) to automatically determine the semantic purpose of the functions in a parser.

## Quickstart: Docker

The easiest way to run PolyTracker is via Docker. To build the Docker
container, simply run the following from the root of this repository:
```
docker build -t trailofbits/polytracker .
```

This will create a Docker container with PolyTracker built, and the `CC` environment variable set to `polyclang`. Simply add the code to be instrumented to this container, and as long as its build process honors the `CC` environment variable, the resulting binary will be instrumented.

For a demo of PolyTracker running on the [MuPDF](https://mupdf.com/) parser run this command:
```
docker build -t trailofbits/polytracker-demo -f Dockerfile-mupdf.demo .
```
This creates a Docker container with an instrumented version of MuPDF. You can execute the instrumented version of `mutool` from your host system using some Docker magic provided in the `mutool.sh` script. Simply run that script on any PDF in the current directory and it will run the instrumented version of `mutool draw` on it, saving the JSON output to `polytracker.json` in the current directory.

## Dependencies and Prerequisites

PolyTracker has only been tested on x86\_64 Linux. (Notably, the [DataFlow Sanitizer](https://clang.llvm.org/docs/DataFlowSanitizer.html) that PolyTracker builds upon _does not_ work on macOS.)

The following tools and libraries are required to run PolyTracker:
* LLVM version 7 or 7.1; other later versions may work but have not been tested. The builds in the official Ubuntu Bionic repository appear to be broken; we suggest building LLVM from source or installing it from the official LLVM repositories

## Building PolyTracker from Source

NOTE: While you can build PolyTracker from source, at the moment it only runs in Docker, this will be fixed soon

The following tools are required to build PolyTracker:
* [CMake](https://cmake.org)
* [Ninja](https://ninja-build.org) (`ninja-build` on Ubuntu)
* Python 3.7 and `pip`, for testing purposes (`apt-get -y install python3.7 python3-pip`)

First, make sure that the LLVM 7 binaries have priority in your `PATH`, _e.g._,
```
export PATH="/usr/lib/llvm-7/bin:${PATH}"
```
Next, from the root directory of this repository, run
```
mkdir build && cd build
cmake -G Ninja -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++ .. && ninja install
```

This builds and installs two compilers: `polyclang` and
`polyclang++`. These are wrappers around `clang` and `clang++`,
respectively, and will add the PolyTracker instrumentation.

## Instrumenting a Program with PolyTracker

All that is required is to modify the program's build system to use `polyclang`/`polyclang++` instead of its default compiler. The easiest way to do this is to set the compiler environment variables to them:
```
export CC=`which polyclang`
export CXX=`which polyclang++`
```

For example, let's work through how to build MuPDF with PolyTracker instrumentation:

```
git clone --recursive git://git.ghostscript.com/mupdf.git
cd mupdf
git submodule update --init
make -j10 HAVE_X11=no HAVE_GLUT=no prefix=./bin install
``` 

Or if you would like to build the debug version, as we do in our Dockerfile:

```
make -j10 HAVE_X11=no HAVE_GLUT=no prefix=./bin debug
```

## Enviornment Variables 

PolyTracker accepts configuration paramters in the form of enviornment variables to avoid recompiling target programs. The current enviornment variables PolyTracker supports is: 

```
POLYPATH: The path to the file to mark as tainted 

POLYCACHE: The size (in bytes) of a cache used to increase processing 

POLYTTL: This value is an initial "strength" value for taint nodes, when new nodes are formed, the average is taken. When the TTL value is 0, the node is considered clean 

POLYTIME: This value is the time (in seconds) used to create snapshots. A snapshot contains the taint tracing data so far during execution, if this is not set, no snapshots will be taken. 

POLYDUMP: Instead of dumping json, if this is set to TRUE it will dump the contents of shadow memory to a file. 
```

## Running an Instrumented Program

The PolyTracker instrumentation looks for the `POLYPATH` environment variable to specify which input file's bytes are meant to be tracked. (Note: PolyTracker can in fact track multiple input files—and really any file-like stream such as network sockets—however, we have thus far only exposed the capability to specify a single file. This will be improved in a future release.)

The instrumented software will write its output to `polytracker.json` in the current directory.

For example, with our instrumented version of MuPDF, run
```
POLYPATH=input.pdf POLYTTL=32 POLYCACHE=100000 ./mutool info input.pdf
```
On program exit, `polytracker.json` will be created in the current directory.

Alternatively, if you want to examine the results to process yourself, there is an option to dump the tracked results to disk. Simply set the `POLYDUMP` enviornment variable to `TRUE`. 

```
POLYPATH=input.pdf POLYTTL=10 POLYCACHE=100000 POLYDUMP=TRUE ./mutool info input.pdf
```

This will produce two files, one is the contents of the in memory taint forest, and the other is a json that maps function names to a set of taint nodes touched in a comparison. The schema for the taint forest can be found in `dfsan_types.h`, which is just the node structure that is written to disk.  

## Snapshotting 

With Polytracker is it possible to create a snapshot of processed taint during tracking. The `POLYTIME` variable allows the user to specify an interval at which to take snapshots. These snapshots are stored as jsons in the current working directory. 

```
POLYPATH=input.pdf POLYTIME=1 ./mutool draw input.pdf 
``` 

## Current Status and Known Issues

Taints will not propagate through dynamically loaded libraries unless
those libraries were compiled from source using PolyTracker, _or_
there is specific support for the library calls implemented in
PolyTracker. There _is_ currently support for propagating taint
throught the majority of uninstrumented C standard library calls. 
To be clear, programs that use uninstrumented functions will still run normally,
however, operations performed by unsupported library calls will not
propagate taint. We are currently working on adding robust support for
C++ programs, but currently the best results will be from C programs.

The worst case performance of PolyTracker is exercised when a single
byte in memory is simultaneously tainted by a large number of input
bytes from the source file. This is most common when instrumenting
compression and cryptographic algorithms that have large block
sizes. There are a number of mitigations for this behavior currently
being researched and developed.

## License and Acknowledgements

This research was developed by [Trail of
Bits](https://www.trailofbits.com/) with funding from the Defense
Advanced Research Projects Agency (DARPA) under the SafeDocs program
as a subcontractor to [Galois](https://galois.com). It is licensed
under the [Apache 2.0 lisense](LICENSE). © 2019, Trail of Bits.

## Maintainers
[Carson Harmon](https://github.com/ThatsNotVeryCashMoneyOfYou)<br />
[Evan Sultanik](https://github.com/ESultanik)<br />
[Brad Larsen](https://github.com/bradlarsen)<br />
<br />
`firstname.lastname@trailofbits.com`
