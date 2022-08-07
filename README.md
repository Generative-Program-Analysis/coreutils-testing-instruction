Testing Coreutils
======================

This manual contains detailed instructions on testing GNU's Coreutils.
The testing pipeline depends on *LLVM 11*.

### Step 1: Building the file system linker

We need a lightweight linker to link the Coreutils program with the uClibc library
and the POSIX file system. The linker's implementation extracts all linking and
code optimization related code from KLEE.

#### Install dependencies

Install the basic dependencies:
```bash
$ sudo apt-get install build-essential cmake file g++-multilib gcc-multilib git python3-pip
```

If you are using a recent Ubuntu distribution (e.g. 21.10), install the LLVM-11
packages provided from the package manager.
```bash
$ sudo apt-get install clang-11 llvm-11 llvm-11-dev llvm-11-tools
```

Otherwise, you can [build LLVM-11 from source](https://releases.llvm.org/11.0.1/docs/GettingStarted.html).

#### Build the preprocessor `fs-linker`

1. First, clone the linker repo.
```bash
$ git clone https://github.com/Generative-Program-Analysis/fs-linker.git
$ cd fs-linker
```

2. Configure the linker
```bash
$ mkdir build
$ cd build
$ cmake -DLLVM_CONFIG_BINARY=<ABSOLUTE_PATH_TO_LLVM_CONFIG_11_BINARY> \
        -DCMAKE_INSTALL_PREFIX=<YOUR_INSTALL_PATH_PREFIX> \
        -DCMAKE_BUILD_TYPE=<Release/Debug> ..
```
Note: if you do not wish to install the linker binary, please omit the `-DCMAKE_INSTALL_PREFIX` option.

3. Build the linker
```bash
$ make # or make install
```
The linker binary `fs-linker` will be generated under `build/bin` and `<YOUR_INSTALL_PATH_PREFIX>/bin`
(if user specifies `-DCMAKE_INSTALL_PREFIX`).

#### Usage

The linker combines Coreutils programs with KLEE's POSIX file system (Step 2)
and the uClibc library (Step 3) into a single LLVM IR program.
The linker also performs source code optimization and transformation
on the combined program.

Below are some basic options to use `fs-linker`:
- `--posix-path=<PATH_TO_POSIX_ARCHIVE>`   : this option should be the absolute path of the POSIX file system's archive. If you omit this option, the source program will not be linked with KLEE's POSIX file system but use LLSC's internal file system.
- `--uclibc-path=<PATH_TO_UCLIBC_ARCHIVE>` : this option should be the absolute path of the uClibc library's archive. This option is required if you are linking Coreutils programs.
- `--switch-type=simple`                   : this option will lower the switch instructions to a chain of branch instructions.
- `--optimize`                             : optimize the code using a series of pre-selected optimization passes.
- `-o`                                     : this option specifies the name of the output LLVM IR program.
- To see all options, run ```fs-linker --help```

##### Sample usage

```bash
$ fs-linker --posix-path=${POSIX_SOURCE_DIR}/build/runtime/lib/libllscRuntimePOSIX64.bca \
            --uclibc-path=${UCLIBC_SOURCE_DIR}/lib/libc.a \
            --switch-type=simple \
            ./echo.bc -o echo_linked.ll
```

The command above links input program `echo.bc` with POSIX file system
and the uClibc library, lowers switch instructions
and produces the linked LLVM IR program into `echo_linked.ll`.

### Step 2: Building the POSIX file system

This step builds a modified version of KLEE's POSIX file system model in order
to test Coreutils programs (or any programs using POSIX file system APIs) with
both KLEE and LLSC.

1. First, clone the `posix-runtime` repository:
```bash
$ git clone https://github.com/Generative-Program-Analysis/posix-runtime.git
$ cd posix-runtime
```

2. Configure the POSIX file system
```bash
$ mkdir build
$ cd build
$ cmake -DLLVM_CONFIG_BINARY=<ABSOLUTE_PATH_TO_LLVM_CONFIG_11_BINARY> \
        -DLLVMCC=<ABSOLUTE_PATH_TO_CLANG_11_BINARY> \
        -DLLVMCXX=<ABSOLUTE_PATH_TO_CLANG++_11_BINARY> \
        -DCMAKE_INSTALL_PREFIX=<YOUR_INSTALL_PATH_PREFIX> \
        -DLLSC_HEADER_DIR=<DIR_TO_YOUR_LLSCCLIENT_HEADER> \
        -DKLEE_HEADER_DIR=<DIR_TO_YOUR_KLEE_HEADER> \
        -DRUNTIME_CFLAGS="-O0 -fno-discard-value-names" ..
```
Note: if you do not wish to install the posix file system library, please omit the `-DCMAKE_INSTALL_PREFIX` option.

Note: we will build the POSIX file system model for both KLEE and our engine LLSC. So the user should provide the engine's external API header file, i.e. `-DLLSC_HEADER_DIR` (the directory where `llsc_client.h` is located) and `-DKLEE_HEADER_DIR` (the directory where `klee.h` is located).

Note: we use `-DRUNTIME_CFLAGS` to pass optimization flags to `clang`, which produces bitcode files of source programs.
By default, we disable optimization by passing `-O0 -fno-discard-value-names`.
If you wish to disable assertions, you can pass `-DNDEBUG` in addition.

3. Build the POSIX runtime
```bash
$ make # or make install
```
The archived posix-filesystem `libkleeRuntimePOSIX64.bca` (for KLEE) and  `libllscRuntimePOSIX64.bca` (for LLSC) will be generated under `build/runtime/lib`, and `<YOUR_INSTALL_PATH_PREFIX>/lib` (if user specifies `-DCMAKE_INSTALL_PREFIX`).

### Step 3: Building the uClibc library

1. First, clone the `klee-uclibc` repository:

```bash
$ git clone https://github.com/Generative-Program-Analysis/klee-uclibc.git
$ cd klee-uclibc
$ git checkout evaluation_uclibc_version
```

2. Configure the klee-uClibc library:

```bash
$ ./configure --make-llvm-lib \
              --with-cc <ABSOLUTE_PATH_TO_CLANG_11_BINARY> \
              --with-llvm-config <ABSOLUTE_PATH_TO_LLVM_CONFIG_11_BINARY>
```

Note: The assertions in uClibc library are disabled by default, you may add `--enable-assertions` option to enable assertions.

3. Build the klee-uClibc:

```bash
$ make KLEE_CFLAGS="-fno-discard-value-names -mlong-double-64"
```

Note: we use `KLEE_CFLAGS` to pass additional compiler flags in the end, `-mlong-double-64` is used to disable fp80 datatype.

The generated uClibc library archive is `lib/libc.a`.

# Step 4: Building Coreutils programs

1. First, clone the Coreutils repository:

```bash
$ git clone git://git.sv.gnu.org/coreutils
$ cd coreutils
$ git checkout tags/v8.32 -b <branch_name>
```

Note: we will be testing on Coreutils v8.32 release.

2. Install and setup `wllvm`:

Coreutils programs need to be compiled into LLVM bitcode in order to be executed on KLEE and our engine.
We also need to install `wllvm` to compile multiple modules of source code into a
single LLVM whole-program bitcode file:
```bash
$ sudo pip3 install wllvm
```

To successfully execute `wllvm`, it is necessary to set several environment variables:

```bash
$ export LLVM_COMPILER=clang                         // required, we use llvm bitcode compiler
$ export LLVM_CC_NAME=<your-clang-11-name>           // optional: can be set if your clang compiler is not called clang but something like clang-11
$ export LLVM_CXX_NAME=<your-clang++-11-name>        // optional: same as above
$ export LLVM_LINK_NAME=<your-llvm-link-11-name>     // optional: same as above
$ export LLVM_AR_NAME=<your-llvm-ar-11-name>         // optional: same as above
$ export LLVM_COMPILER_PATH=<YOUR_LLVM_BINARY_PATH>  // optional: can be set to the absolute path to the folder that contains your LLVM 11 binary. This prevents searching for the binary in your PATH environment variable.
```

3. Build coreutils program

Please refer to `README-prereq` under the root directory of coreutils to
install the dependencies required for building coreutils.

Note: For coreutils v8.32, newer Bison versions (>= v3.7) will cause errors, please install older Bison version before building.

To compile coreutils:

```bash
$ ./bootstrap
$ mkdir obj-llvm
$ cd obj-llvm
$ CC=wllvm ../configure \
  --disable-nls \
  CFLAGS="-O0 -Xclang -disable-O0-optnone -fno-discard-value-names -D__NO_STRING_INLINES -D_FORTIFY_SOURCE=0 -U__OPTIMIZE__"
$ make
```

Note: `CFLAGS` specifies the flags for compiling Coreutils. We use `-O0 -Xclang -disable-O0-optnone` to disable more aggressive optimizations. You can also use `-O1 -Xclang -disable-llvm-passes` to enable some optimizations, and the generated bitcode under this option is more suited for optimization later (in the linking stage).

Then we need to extract the LLVM bitcode using `extract-bc` (a `wllvm` utility):

```bash
cd src
find . -executable -type f | xargs -I '{}' extract-bc '{}'
cd ..
```

# Step 5: Testing

After the last step, compiled native binary files (e.g. `echo`, `cat`) and their LLVM bitcode files (e.g. `echo.bc`, `cat.bc`) are located under `obl-llvm/src`.

For testing, we can create a separate folder under `obj-llvm`:
```bash
mkdir playground
cd playground
cp ../src/*.bc ./
```

- Using KLEE

If we want to test the coreutils bitcode (for example echo) on KLEE.

First refer to [KLEE's document](https://klee.github.io/build-llvm11/) to build KLEE.
Then using the `fs-linker` program to produce a single LLVM IR program
containing the POSIX file system (with KLEE's external API) and the uClibc
library.

Using `echo` as example:

```bash
fs-linker --posix-path=${dir_to_posix_archive}/libkleeRuntimePOSIX64.bca \
          --uclibc-path=${dir_to_uclibc_folder}/lib/libc.a \
          ./echo.bc -o echo_klee.ll
```

Since we have already produced the program with the POSIX/uClibc library, we
do not need to specify these options when running KLEE.
We need to use `--switch-type=simple` to lower `switch` instructions into
branches, to make sure that both KLEE and LLSC treats `switch` in the same way:

```bash
klee --switch-type=simple echo_klee.ll --sym-stdout --sym-arg 8
```

KLEE should explore 4971 paths after running the above command. For more details on testing Coreutils programs with KLEE, please refer to [KLEE's document](https://klee.github.io/tutorials/testing-coreutils/).

- Using LLSC with KLEE's POSIX model

To test Coreutils bitcode programs using LLSC, we need to link the program with
the POSIX file system (with LLSC's external API) and the uClibc library again using
the `fs-linker` program:

```bash
fs-linker --posix-path=${dir_to_posix_archive}/libllscRuntimePOSIX64.bca \
          --uclibc-path=${dir_to_uclibc_folder}/lib/libc.a \
          --switch-type=simple \
          ./echo.bc -o echo_linked.ll
```

Then run the linked IR in LLSC with the following Scala code in the engine:
```bash
testLLSC(new ImpCPSLLSC, List(TestPrg(echo_linked, "echo_linked_posix", "@main", testcoreutil, Seq("--cons-indep","--argv=./echo.bc --sym-stdout --sym-arg 8"), nPath(4971)++status(0))))
```
You should observe same path number 4971.

- Using LLSC without POSIX

To test Coreutils bitcode files using LLSC with its the internal file system,
we only need to link the program with the uClibc library.

```bash
fs-linker --uclibc-path=${dir_to_uclibc_folder}/lib/libc.a \
          --switch-type=simple \
          ./echo.bc -o echo_llsc_linked.ll
```

Then run the linked IR program with the following Scala code in the engine:

```bash
testLLSC(new ImpCPSLLSC, List(TestPrg(echo_llsc_linked, "echo_llsc_linked", "@main", testcoreutil, Seq("--cons-indep","--argv=./echo.bc #{8}"), nPath(4971)++status(0))))
```

You should observe same path number 4971.

#### Measuring coverage with `gcov`

This feature is still under development for LLSC. Please refer to [Testing Coreutils](https://klee.github.io/tutorials/testing-coreutils/) about testing with `gcov` in KLEE.
