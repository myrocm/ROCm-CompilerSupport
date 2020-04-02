Code Object Manager (Comgr)
===========================

The Comgr library provides APIs for compiling and inspecting AMDGPU code
objects. The API is documented in the [header file](include/amd_comgr.h).

Building the Code Object Manager
--------------------------------

Comgr depends on [LLVM](https://github.com/RadeonOpenCompute/llvm-project) and
[AMDDeviceLibs](https://github.com/RadeonOpenCompute/ROCm-Device-Libs). One way
to make these visible to the Comgr build process is by setting the
`CMAKE_PREFIX_PATH` to include either the build directory or install prefix of
each of these components, separated by a semicolon. Both should be built using
either sources with the same ROCm release tag, or from the `amd-stg-open`
branch. LLVM should be built with at least
`LLVM_ENABLE_PROJECTS='llvm;clang;lld'` and
`LLVM_TARGETS_TO_BUILD='AMDGPU;X86'`.

An example `bash` session to build Comgr on Linux using GNUMakefiles is:

    $ LLVM_PROJECT=~/llvm-project
    $ DEVICE_LIBS=~/device-libs
    $ COMGR=~/support/lib/comgr
    $ mkdir -p "$LLVM_PROJECT/build"
    $ cd "$LLVM_PROJECT/build"
    $ cmake \
        -DCMAKE_BUILD_TYPE=Release \
        -DLLVM_ENABLE_PROJECTS="llvm;clang;lld" \
        -DLLVM_TARGETS_TO_BUILD="AMDGPU;X86" \
        ../llvm
    $ make
    $ mkdir -p "$DEVICE_LIBS/build"
    $ cd "$DEVICE_LIBS/build"
    $ cmake \
        -DCMAKE_BUILD_TYPE=Release \
        -DLLVM_DIR="$LLVM_PROJECT/build" \
        ..
    $ make
    $ mkdir -p "$COMGR/build"
    $ cd "$COMGR/build"
    $ cmake \
        -DCMAKE_BUILD_TYPE=Release \
        -DCMAKE_PREFIX_PATH="$LLVM_PROJECT/build;$DEVICE_LIBS/build" \
        ..
    $ make
    $ make test

The equivalent on Windows in `cmd.exe` using Visual Studio project files is:

    > set LLVM_PROJECT="%HOMEPATH%\llvm-project"
    > set DEVICE_LIBS="%HOMEPATH%\device-libs"
    > set COMGR="%HOMEPATH%\support\lib\comgr"
    > mkdir "%LLVM_PROJECT%\build"
    > cd "%LLVM_PROJECT%\build"
    > cmake ^
        -DLLVM_ENABLE_PROJECTS="llvm;clang;lld" ^
        -DLLVM_TARGETS_TO_BUILD="AMDGPU;X86" ^
        ..\llvm
    > msbuild /p:Configuration=Release ALL_BUILD.vcxproj
    > mkdir "%DEVICE_LIBS%\build"
    > cd "%DEVICE_LIBS%\build"
    > cmake ^
        -DLLVM_DIR="%LLVM_PROJECT%\build" ^
        ..
    > msbuild /p:Configuration=Release ALL_BUILD.vcxproj
    > mkdir "%COMGR%\build"
    > cd "%COMGR%\build"
    > cmake ^
        -DCMAKE_PREFIX_PATH="%LLVM_PROJECT%\build;%DEVICE_LIBS%\build" ^
        ..
    > msbuild /p:Configuration=Release ALL_BUILD.vcxproj
    > msbuild /p:Configuration=Release RUN_TESTS.vcxproj

Optionally,
[AddressSanitizer](https://github.com/google/sanitizers/wiki/AddressSanitizer)
may be enabled during development via `-DENABLE_ASAN=On` during the Comgr
`cmake` step.

Depending on the Code Object Manager
------------------------------------

Comgr exports a CMake package named `amd_comgr` for both the build and install
trees. This package defines a library target named `amd_comgr`. To depend on
this in your CMake project, use `find_package`:

    find_package(amd_comgr REQUIRED CONFIG)
    ...
    target_link_libraries(your_target amd_comgr)

If Comgr is not installed to a standard CMake search directory, the path to the
build or install tree can be supplied to CMake via `CMAKE_PREFIX_PATH`:

    cmake -DCMAKE_PREFIX_PATH=path/to/comgr/build/or/install

Debugging the Code Object Manager
---------------------------------

Comgr supports some environment variables to aid in debugging. These include:

* `AMD_COMGR_SAVE_TEMPS`: If this is set, and is not "0", Comgr does not delete
  temporary files generated during compilation. These files do not appear in
  the current working directory, but are instead left in a platform-specific
  temporary directory (typically `/tmp` on Linux and `C:\Temp` or the path
  found in the `TEMP` environment variable on Windows).
* `AMD_COMGR_REDIRECT_LOGS`: If this is not set, or is set to "0", logs are
  returned to the caller as normal. If this is set to "stdout"/"-" or "stderr",
  logs are instead redirected to the standard output or error stream,
  respectively. If this is set to any other value, it is interpreted as a
  filename which logs should be appended to.
* `AMD_COMGR_EMIT_VERBOSE_LOGS`: If this is set, and is not "0", logs will
  include additional Comgr-specific informational messages.

Versioning
----------

Comgr is versioned according to a `major.minor` number scheme. The version of
the library can be determined dynamically via the `amd_comgr_get_version`
function. The version is not changed due to bug-fixes. The minor version number
is incremented for each backwards-compatible change introduced. The major
version number is incremented, and the minor version is reset to zero, for each
backwards-incompatible change introduced.

* `1.6`: Add `AMD_COMGR_SYMBOL_TYPE_AMDGPU_HSA_KERNEL` for Code Object V2
  kernel symbols.
* `1.5`: Add `AMD_COMGR_SYMBOL_TYPE_UNKNOWN` for unknown/unsupported ELF symbol
  types. This fixes a bug where these symbols were previously reported as
  `AMD_COMGR_SYMBOL_TYPE_NOTYPE`.
* `1.4`: Support out-of-process HIP compilation to fat binary.
* `1.3`: Introduce `amd_comgr_action_info_set_option_list`,
  `amd_comgr_action_info_get_option_list_count`, and
  `amd_comgr_action_info_get_option_list_item` to replace the old option APIs
  `amd_comgr_action_info_set_options` and `amd_comgr_action_info_get_options`.
  The old APIs do not support arguments with embedded delimiters, and are
  replaced with an array-oriented API. The old APIs are deprecated and will be
  removed in a future version of the library.
* `1.2`: Introduce `amd_comgr_disassemble_instruction` and associated APIS.
* `1.1`: First versioned release. Versions before this have no guaranteed
  compatibility.

Coding Standards
----------------

Wherever possible, Comgr adheres to the same coding standards as
[LLVM](https://llvm.org/docs/CodingStandards.html). Comgr also includes
configuration files for
[clang-format](https://clang.llvm.org/docs/ClangFormat.html) and
[clang-tidy](https://clang.llvm.org/extra/clang-tidy/), which should be used to
ensure patches conform where reasonable.

One notable exception is the `test/` subdirectory which prefers `camelBack` for
identifiers rather than `CamelCase`.
