# Testsuite using CMake

This Folder contains a CMake Project for the Testsuite, it should
be easy to open in a variety of IDEs including

-   CLion
-   VSCode with assorted CMake Extensions
-   Eclipse with `cmake4eclipse` Plugin
-   QTCreator

Many tools integrate directly with CMake or can depend on a `compile_commands.json`
database that will be generated after adding `-DCMAKE_EXPORT_COMPILE_COMMANDS=ON`
as option to CMake.

The toolchain can be added as single parameter, the project can run the Tests
on Linux if wine is available.

## Build per Commandline

The commandline Tools

``` bash
# Configure
cmake --toolchain /opt/llvm-mingw/share/cmake/llvm-mingw_toolchainfile.cmake -S test/cmake/tests -B /tmp/build
# Build
cmake --build /tmp/build
# Run Tests
ctest --test-dir /tmp/build
# Install + package
cmake --install /tmp/build --prefix /tmp/install
tar c -f /tmp/archive.tar -C /tmp/install .
```

## VSCode

[VSCode](https://code.visualstudio.com/) is a free editor, has good
support for CMake via extensions and once configured can simply open
this or any other CMake project directly.

Some one-time global configuration is necessary, best done in the
[User Settings](https://code.visualstudio.com/docs/getstarted/settings#_settingsjson),
an exemplary config for Linux, paths to `clangd` and `lldb-dap`
might need to be adjusted:

``` json
{
    "cmake.copyCompileCommands": "${workspaceFolder}/build/compile_commands.json",
    "cmake.buildDirectory": "${workspaceFolder}/_build",
    "cmake.preferredGenerators": [
        "Ninja",
        "Unix Makefiles"
    ],
    "cmake.configureArgs": [
        "-DCMAKE_EXPORT_COMPILE_COMMANDS=ON"
    ],
    "clangd.path": "clangd",
    "clangd.arguments": [
        "--clang-tidy"
    ],
    "lldb-dap.executable-path": "/usr/bin/lldb-dap-18"
}
```

After creating a configuration for the LLVM-Mingw toolchain,
the project should compile and the tests should be available.

If `clangd` is setup correctly, [rich information](https://clangd.llvm.org/features)
about used classes and code-completion should be available.

`lldb-dap` brings debugging support, but this only works native
(atleast without some tinkering).
