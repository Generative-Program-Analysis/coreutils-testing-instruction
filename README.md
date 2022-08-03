Coreutils-testpipeline
======================

This manual contains detailed instruction on building coreutils test pipeline

Note: We use LLVM 11 for the entire test-pipeline build
=======================================================

# Step 1: Building filesystem linker

We need a lightweight linker to link the coreutils program with uclibc library and POSIX filesystem. The linker extracts all linking and code optimization related code from KLEE.
## Dependencies
```bash
$ sudo apt-get install build-essential cmake file g++-multilib gcc-multilib git python3-pip
```
### Install wllvm to make it easier to compile programs to LLVM bitcode
```bash
$ sudo pip3 install wllvm
```
### Installing LLVM-11
The linker like its origin KLEE is based on LLVM. We use LLVM-11 tool chain for build.

If you are using a recent Ubuntu (e.g. 21.10) use the LLVM packages provided by LLVM itself.
```bash
$ sudo apt-get install clang-11 llvm-11 llvm-11-dev llvm-11-tools
```
otherwise, you can build LLVM-11 from source, please refer to [LLVM Getting Started](https://releases.llvm.org/11.0.1/docs/GettingStarted.html)

## Building
1. First, clone the linker repo.
```bash
$ git clone https://github.com/Generative-Program-Analysis/fs-linker.git
$ cd fs-linker
```
2. Configure the linker
```bash
$ mkdir build
$ cd build
$ cmake -DLLVM_CONFIG_BINARY=<ABSOLUTE_PATH_TO_LLVM_CONFIG_11_BINARY>  -DCMAKE_INSTALL_PREFIX=<YOUR_INSTALL_PATH_PREFIX>  -DCMAKE_BUILD_TYPE=<Release/Debug> ..
```
Note: if you do not wish to install the linker binary, please omit the `-DCMAKE_INSTALL_PREFIX` option.

3. Build the linker
```bash
$ make install / make
```
The generated linker binary `fs-linker` will be generated under `build/bin`, and `<YOUR_INSTALL_PATH_PREFIX>/bin` (if user specifies `-DCMAKE_INSTALL_PREFIX`).

## Usage
This linker can link coreutils programs with KLEE's POSIX filesystem (Step 2) and KLEE's uClibc library (Step 3) and perform some code optimization and transformation.

Below are some basic options:
- `--posix-path=<PATH_TO_POSIX_ARCHIVE>`   : this option should be the absolute path of your POSIX filesystem's archive. If you omit this option, the source program will not be linked to KLEE's POSIX filesystem but use LLSC's internal filesystem.
- `--uclibc-path=<PATH_TO_UCLIBC_ARCHIVE>` : this option should be the absolute path of your uClibc library's archive. This option is required if you are linking coreutils programs.
- `--switch-type=simple`                   : this option will lower the switch inst in the input IR to a chained branch instructions.
- `--optimize`                             : optimize the code using a series of pre-selected optimization passes.
- `-o`                                     : this option specify the name of the output LLVM IR.
- To see all options run``` fs-linker --help```

### Sample usage
```bash
$ fs-linker --posix-path=${POSIX_SOURCE_DIR}/build/runtime/lib/libllscRuntimePOSIX64.bca --uclibc-path=${UCLIBC_SOURCE_DIR}/lib/libc.a  --switch-type=simple   ./echo.bc -o echo_linked.ll
```
The command above link input program `echo.bc` with POSIX filesystem (with LLSC's external function) and uClibc library, lower the switch instructions and output the linked IR into `echo_linked.ll`.



# Step 2: Building POSIX filesystem
This is a modified version of KLEE's POSIX filesystem in order to run on both KLEE and our engine.
1. First, clone the posix-runtime repo.
```bash
$ git clone https://github.com/Generative-Program-Analysis/posix-runtime.git
$ cd posix-runtime
```
2. Configure the posix filesystem
```bash
$ mkdir build
$ cd build
$ cmake -DLLVM_CONFIG_BINARY=<ABSOLUTE_PATH_TO_LLVM_CONFIG_11_BINARY>  -DLLVMCC=<ABSOLUTE_PATH_TO_CLANG_11_BINARY> -DLLVMCXX=<ABSOLUTE_PATH_TO_CLANG++_11_BINARY>  -DCMAKE_INSTALL_PREFIX=<YOUR_INSTALL_PATH_PREFIX>  -DLLSC_HEADER_DIR=<DIR_TO_YOUR_LLSCCLIENT_HEADER> -DKLEE_HEADER_DIR=<DIR_TO_YOUR_KLEE_HEADER> -DRUNTIME_CFLAGS="-O0  -fno-discard-value-names"  ..
```
Note1: if you do not wish to install the posix filesystem library, please omit the `-DCMAKE_INSTALL_PREFIX` option.

Note2: we will be building posix flesystem for both KLEE and our engine LLSC. So user should provide the engine's external api header,  `-DLLSC_HEADER_DIR` (the directory where `llsc_client.h` is located) and `-DKLEE_HEADER_DIR` (the directory where `klee.h` is located).

Note3: we use `-DRUNTIME_CFLAGS` to pass optimization flags to the clang bitcode compiler. By default, we disable optimization by passing `-O0  -fno-discard-value-names`. If you wish to disable assert, you can pass `-DNDEBUG` in addition.

3. Build the posix runtime
```bash
$ make install / make
```
The archived posix-filesystem `libkleeRuntimePOSIX64.bca` (for KLEE) and  `libllscRuntimePOSIX64.bca` (for LLSC) will be generated under `build/runtime/lib`, and `<YOUR_INSTALL_PATH_PREFIX>/lib` (if user specifies `-DCMAKE_INSTALL_PREFIX`).

# Step 3: Building uClibc library
1. First, clone the klee-uClibc repo.
```bash
$ git clone https://github.com/Generative-Program-Analysis/klee-uclibc.git
$ cd klee-uclibc
$ git checkout evaluation_uclibc_version
```
2. Configure the klee-uClibc library
```bash
$ ./configure --make-llvm-lib --with-cc <ABSOLUTE_PATH_TO_CLANG_11_BINARY> --with-llvm-config <ABSOLUTE_PATH_TO_LLVM_CONFIG_11_BINARY>
```
Note: The assertions in uClibc library are disabled by default, you may add `--enable-assertions` option to enable assertions.

3. Build the klee-uClibc
```bash
$ make KLEE_CFLAGS=" -fno-discard-value-names -mlong-double-64"
```
Note: we use `KLEE_CFLAGS` to pass additional compiler flags in the end, `-mlong-double-64` is used to disable fp80 datatype.

The generated uClibc library archive is `lib/libc.a`.

# Step 4: Building coreutils program
1. First, clone the coreutils repo.
```bash
$ git clone git://git.sv.gnu.org/coreutils
$ cd coreutils
$ git checkout tags/v8.32 -b <branch_name>
```
Note: we will be testing on coreutils v8.32 release

2. `wllvm` setup

Coreutils programs need to be compiled into LLVM bitcode in order to be executed on KLEE and our engine. So we will need to use the previously installed `wllvm` (in Step 1) to compile both the native binary and LLVM bitcode.

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

### Dependencies
Please refer to `README-prereq` under the root directory of coreutils to install the dependencies required for building coreutils.

Note: For coreutils v8.32, new Bison versions (>= v3.7) will cause error, please install older Bison version before building.

### Building

```bash
$ ./bootstrap
$ mkdir obj-llvm
$ cd obj-llvm
$ CC=wllvm ../configure --disable-nls CFLAGS="-O0 -Xclang -disable-O0-optnone -fno-discard-value-names -D__NO_STRING_INLINES  -D_FORTIFY_SOURCE=0 -U__OPTIMIZE__"
$ make
```
Note: `CFLAGS` specifies the compilation falgs for compiling coreutils. We use ```-O0 -Xclang -disable-O0-optnone``` to disable optimizations. You can also use ```-O1 -Xclang -disable-llvm-passes``` to disable optimizations, the generated bitcode under this option is more suited for optimization later (in linking stage).

Then we need to extract the LLVM bitcode using `extract-bc` (a `wllvm` utility).

```bash
cd src
find . -executable -type f | xargs -I '{}' extract-bc '{}'
cd ..
```

# Step 5: Testing

The compiled native binary (echo, cat ...) and LLVM bitcode (echo.bc, cat.bc ...) will be located under `obl-llvm/src`.

For testing, we can create a separate folder under `obj-llvm`
```bash
mkdir playground
cd playground
cp ../src/*.bc ./
```
- KLEE

If we want to test the coreutils bitcode (for example echo) on KLEE.

First refer to [build KLEE](https://klee.github.io/build-llvm11/) to build KLEE, then link the program with POSIX filesystem (with KLEE's external api) and uClibc library.
```bash
fs-linker --posix-path=${dir_to_posix_archive}/libkleeRuntimePOSIX64.bca --uclibc-path=${dir_to_uclibc_folder}/lib/libc.a  ./echo.bc -o echo_klee.ll
```
Then execute the linked IR. Since KLEE has an internal strategies for executing switch instruction which will cause the path number to be lower than our engine. We use `--switch-type=simple` to lower switch into branches to make sure that we can observe same path number for KLEE and our engine LLSC.
```bash
klee --switch-type=simple echo_klee.ll --sym-stdout --sym-arg 8
```
KLEE should explore 4971 paths for the above command. For more details on running coreutils programs on KLEE, please refer to [Testing Coreutils](https://klee.github.io/tutorials/testing-coreutils/).

- LLSC with POSIX

If we want to test the coreutils bitcode (for example echo) on LLSC linked with POSIX filesystem.

Link the program with POSIX filesystem (with LLSC's external api) and uClibc library.

```bash
fs-linker --posix-path=${dir_to_posix_archive}/libllscRuntimePOSIX64.bca --uclibc-path=${dir_to_uclibc_folder}/lib/libc.a --switch-type=simple ./echo.bc -o echo_linked.ll
```

Then run the linked IR in LLSC with the following config in our engine
```bash
testLLSC(new ImpCPSLLSC, List(TestPrg(echo_linked, "echo_linked_posix", "@main", testcoreutil, Seq("--cons-indep","--argv=./echo.bc --sym-stdout --sym-arg 8"), nPath(4971)++status(0))))
```
You should observe same path number 4971.


- LLSC without POSIX

If we want to test the coreutils bitcode (for example echo) on LLSC using the internal filesystem.

Link the program with only uClibc library.

```bash
fs-linker --uclibc-path=${dir_to_uclibc_folder}/lib/libc.a --switch-type=simple ./echo.bc -o echo_llsc_linked.ll
```

Then run the linked IR in LLSC with the following config in our engine
```bash
testLLSC(new ImpCPSLLSC, List(TestPrg(echo_llsc_linked, "echo_llsc_linked", "@main", testcoreutil, Seq("--cons-indep","--argv=./echo.bc #{8}"), nPath(4971)++status(0))))
```
You should observe same path number 4971.

## Measuring coverage with gcov

This feature is still under development for our engine. Please refer to [Testing Coreutils](https://klee.github.io/tutorials/testing-coreutils/) about testing with gcov in KLEE.