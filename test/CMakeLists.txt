cmake_minimum_required(VERSION 3.17..3.30)

# compatibility for MSVC toolchains
if(POLICY CMP0141)
  cmake_policy(SET CMP0141 NEW)
endif()

if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  message(STATUS "Setting build type to 'Debug' as none was specified.")
  set(CMAKE_BUILD_TYPE Debug CACHE
          STRING "Choose the type of build." FORCE)
endif()

if(NOT CMAKE_HOST_WIN32 AND NOT CMAKE_CROSSCOMPILING_EMULATOR)
  find_program(LLVM_MINGW_WINE_PATH NAMES wine wine64)
  if(LLVM_MINGW_WINE_PATH)
    set(CMAKE_CROSSCOMPILING_EMULATOR "${LLVM_MINGW_WINE_PATH}")
  endif()
endif()

project(test
        VERSION 8.7.1
)

if(NOT MSVC)
  # statically link libraries for an easier time running executables
  # This refers to the C++ library and compiler runtime - libc++ libunwind for LLVM-Mingw
  add_link_options($<$<COMPILE_LANGUAGE:CXX>:-static-libstdc++$<SEMICOLON>-static-libgcc>)
endif()

if(CMAKE_VERSION VERSION_GREATER_EQUAL "3.20")
  cmake_path(GET CMAKE_C_COMPILER PARENT_PATH hint_compiler_path)
else()
  get_filename_component(hint_compiler_path ${CMAKE_C_COMPILER} DIRECTORY)
endif()

find_program(WIDL_COMPILER NAMES ${CMAKE_SYSTEM_PROCESSOR}-w64-mingw32-widl HINTS ${hint_compiler_path})

if(NOT MSVC)
  file(GLOB_RECURSE maybe_dlls
          "${CMAKE_SYSROOT}/*/libomp.dll"
          "${CMAKE_SYSROOT}/*/libc++.dll"
          "${CMAKE_SYSROOT}/*/libunwind++.dll"
          "${CMAKE_SYSROOT}/*/libclang_rt.asan_dynamic-*.dll" "${hint_compiler_path}/../lib/clang/*/libclang_rt.asan_dynamic-*.dll")
  if(maybe_dlls)
    file(COPY ${maybe_dlls}
            DESTINATION ${CMAKE_CURRENT_BINARY_DIR}
    )
  endif()
endif()

# inspired from https://github.com/reactos/reactos/blob/master/sdk/cmake/widl-support.cmake
function(add_idl_headers cmake_target)
  foreach(idlfile ${ARGN})
    if(CMAKE_VERSION VERSION_GREATER_EQUAL "3.20")
      cmake_path(GET idlfile STEM fname)
    else()
      get_filename_component(fname "${idlfile}" NAME_WE)
    endif()
    set(out_header "${CMAKE_CURRENT_BINARY_DIR}/${fname}.h")
    add_custom_command(
            OUTPUT "${out_header}"
            COMMAND ${WIDL_COMPILER} -h -o "${out_header}" ${idlfile}
            DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/${FILE}"
            WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR})
    list(APPEND all_headers ${out_header})
  endforeach()
  add_custom_target(${cmake_target} DEPENDS ${all_headers})
endfunction()

include(CheckCSourceCompiles)
include(CheckCCompilerFlag)

set(ATOMIC_TEST_CODE "#include <stdatomic.h>
struct large {
    int data[20];
};
_Atomic struct large value;
int main() {
    struct large myValue = atomic_load(&value);
    return atomic_is_lock_free(&value);
}")

check_c_source_compiles("${ATOMIC_TEST_CODE}" LLVM_MINGW_ATOMIC_NOLIB)
if(NOT LLVM_MINGW_NO_ATOMIC_LIB)
  set(CMAKE_REQUIRED_LIBRARIES atomic)
  check_c_source_compiles("${ATOMIC_TEST_CODE}" LLVM_MINGW_ATOMIC_WITHLIB)
  unset(CMAKE_REQUIRED_LIBRARIES)
endif()

check_c_source_compiles("#include <corecrt.h>
#if __MSVCRT_VERSION__ < 0x1400 && !defined(_UCRT)
#error not ucrt
#endif
int main() {return 0;}" LLVM_MINGW_HAVE_UCRT)

set(CMAKE_REQUIRED_FLAGS -mguard=cf)
# see llvm/lib/Transforms/CFGuard/CFGuard.cpp
check_c_source_compiles("extern void __guard_dispatch_icall_fptr();
extern void __guard_check_icall_fptr();
#if defined(__x86_64__) || defined(_M_X64)
int main() {return &__guard_dispatch_icall_fptr != 0;}
#else
int main() {return &__guard_check_icall_fptr != 0;}
#endif
" LLVM_MINGW_HAVE_CFGUARD)
unset(CMAKE_REQUIRED_FLAGS)

check_c_compiler_flag(-fno-auto-import LLVM_MINGW_HAVE_AUTOIMPORT)
check_c_compiler_flag(-fopenmp=libomp LLVM_MINGW_HAVE_LIBOMP)
if(CMAKE_C_COMPILER_ID MATCHES "Clang")
  # check_c_compiler_flag(-fstack-protector-strong LLVM_MINGW_HAVE_STACK_PROTECTOR)
  set(LLVM_MINGW_HAVE_STACK_PROTECTOR ON)
endif()

set(CMAKE_REQUIRED_LINK_OPTIONS -fsanitize=address -fsanitize=undefined -fno-sanitize-recover=all)
check_c_compiler_flag("-fsanitize=address -fsanitize=undefined -fno-sanitize-recover=all" LLVM_MINGW_HAVE_SANITIZERS)
unset(CMAKE_REQUIRED_LINK_OPTIONS)

if(LLVM_MINGW_HAVE_AUTOIMPORT)
  # This will result in warnings if you accidentally auto-import
  add_compile_options(-fno-auto-import)
endif()

# TESTS_C
add_executable(hello
        ${PREFIX_PATH}hello.c
)
add_executable(hello-tls
        ${PREFIX_PATH}hello-tls.c
)
add_executable(crt-test
        ${PREFIX_PATH}crt-test.c
)
add_executable(setjmp
        ${PREFIX_PATH}setjmp.c
)

if(LLVM_MINGW_HAVE_AUTOIMPORT)
  # TESTS_C_DLL TESTS_C_LINK_DLL
  add_library(autoimport-lib SHARED
          ${PREFIX_PATH}autoimport-lib.c
  )
  target_compile_options(autoimport-lib PUBLIC -fauto-import)

  add_executable(autoimport-main
          ${PREFIX_PATH}autoimport-main.c
  )
  target_link_libraries(autoimport-main PRIVATE autoimport-lib)
endif()

# TESTS_C_NO_BUILTIN
add_executable(crt-test-no-builtin
        ${PREFIX_PATH}crt-test.c
)
target_compile_options(crt-test-no-builtin PRIVATE -fno-builtin)

# TESTS_C_ANSI_STDIO
add_executable(crt-test-ansi-stdio
        ${PREFIX_PATH}crt-test.c
)
target_compile_definitions(crt-test-ansi-stdio PRIVATE __USE_MINGW_ANSI_STDIO=1)

# TESTS_CPP
add_executable(hello-cpp
        ${PREFIX_PATH}hello-cpp.cpp
)
add_executable(global-terminate
        ${PREFIX_PATH}global-terminate.cpp
)
add_executable(tlstest-main
        ${PREFIX_PATH}tlstest-main.cpp
)
add_executable(longjmp-cleanup
        ${PREFIX_PATH}longjmp-cleanup.cpp
)

# TESTS_CPP_EXCEPTIONS
add_executable(hello-exception
        ${PREFIX_PATH}hello-exception.cpp
)
add_executable(exception-locale
        ${PREFIX_PATH}exception-locale.cpp
)
add_executable(exception-reduced
        ${PREFIX_PATH}exception-reduced.cpp
)

# TESTS_CPP_STATIC
add_executable(hello-exception-static
        ${PREFIX_PATH}hello-exception.cpp
)
target_link_options(hello-exception-static PRIVATE -static)

# TESTS_CPP_DLL TESTS_CPP_LINK_DLL
add_library(tlstest-lib SHARED
        ${PREFIX_PATH}tlstest-lib.cpp
)
#  default would be libtlstest-lib.dll
set_target_properties(tlstest-lib PROPERTIES PREFIX "")

if(CMAKE_C_COMPILER_ID MATCHES "Clang")
  add_library(throwcatch-lib SHARED
          ${PREFIX_PATH}throwcatch-lib.cpp
  )
  add_executable(throwcatch-main
          ${PREFIX_PATH}throwcatch-main.cpp
  )
  target_link_libraries(throwcatch-main PRIVATE throwcatch-lib)
endif()

if(LLVM_MINGW_HAVE_STACK_PROTECTOR)
  # TESTS_SSP
  add_executable(stacksmash
          ${PREFIX_PATH}stacksmash.c
  )
  target_compile_options(stacksmash PRIVATE -fstack-protector-strong)
endif()

# TESTS_FORTIFY
add_executable(bufferoverflow-fortify
        ${PREFIX_PATH}bufferoverflow.c
)
target_compile_definitions(bufferoverflow-fortify PRIVATE _FORTIFY_SOURCE=2)
add_executable(crt-test-fortify
        ${PREFIX_PATH}crt-test.c
)
target_compile_definitions(crt-test-fortify PRIVATE _FORTIFY_SOURCE=2)

if(LLVM_MINGW_HAVE_SANITIZERS)
  # TESTS_ASAN
  add_executable(stacksmash-asan
          ${PREFIX_PATH}stacksmash.c
  )
  target_compile_options(stacksmash-asan PRIVATE -fsanitize=address)
  target_link_options(stacksmash-asan PRIVATE -fsanitize=address)

  # TESTS_UBSAN
  add_executable(ubsan
          ${PREFIX_PATH}ubsan.c
  )
  target_compile_options(ubsan PRIVATE -fsanitize=undefined -fno-sanitize-recover=all)
  target_link_options(ubsan PRIVATE -fsanitize=undefined -fno-sanitize-recover=all)
endif()

if(LLVM_MINGW_HAVE_LIBOMP)
  # TESTS_OMP
  add_executable(hello-omp
          ${PREFIX_PATH}hello-omp.c
  )
  target_compile_options(hello-omp PRIVATE -fopenmp=libomp)
  target_link_options(hello-omp PRIVATE -fopenmp=libomp)
endif()

# TESTS_UWP
add_executable(uwp-error-mingw32
        ${PREFIX_PATH}uwp-error.c
)

# TESTS_IDL
if(WIDL_COMPILER)
  add_executable(idltest
          ${PREFIX_PATH}idltest.c
  )

  add_idl_headers(idltest_headers ${PREFIX_PATH}idltest.idl)
  target_include_directories(idltest PRIVATE
          $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>)
  add_dependencies(idltest idltest_headers)
endif()

# TESTS_RES
add_executable(hello-res
        ${PREFIX_PATH}hello-res.c
        ${PREFIX_PATH}hello-res-rc.rc
)

if(LLVM_MINGW_ATOMIC_WITHLIB OR LLVM_MINGW_ATOMIC_NOLIB)
  # TESTS_ATOMIC
  add_executable(atomic-helpers
          ${PREFIX_PATH}atomic-helpers.c
  )
  target_compile_options(atomic-helpers PRIVATE -Wno-atomic-alignment)
  if(LLVM_MINGW_ATOMIC_WITHLIB)
    target_link_libraries(atomic-helpers PRIVATE atomic)
  endif()
endif()

# TESTS_OTHER_TARGETS
add_executable(hello-mingw32uwp
        ${PREFIX_PATH}hello.c
)
add_executable(hello-tls-mingw32uwp
        ${PREFIX_PATH}hello-tls.c
)

# TESTS_CFGUARD
if(LLVM_MINGW_HAVE_CFGUARD)
  add_executable(cfguard-test
          ${PREFIX_PATH}cfguard-test.c
  )
  target_compile_options(cfguard-test PRIVATE -mguard=cf)
  target_link_options(cfguard-test PRIVATE -mguard=cf)
endif()

if(LLVM_MINGW_HAVE_CFGUARD AND LLVM_MINGW_HAVE_SANITIZERS)
  # TESTS_ASAN_CFGUARD
  add_executable(stacksmash-asan-cfguard
          ${PREFIX_PATH}stacksmash.c
  )
  target_compile_options(stacksmash-asan-cfguard PRIVATE -fsanitize=address -g -mguard=cf)
  target_link_options(stacksmash-asan-cfguard PRIVATE -fsanitize=address -g -mguard=cf)
endif()

get_property(_targets DIRECTORY PROPERTY BUILDSYSTEM_TARGETS)
foreach(_target IN LISTS _targets)
  get_target_property(_type ${_target} TYPE)
  if(_type STREQUAL "EXECUTABLE")
    add_test(NAME ${_target} COMMAND ${_target})
  endif()
  # Need to remove stuff like targets from add_custom_target
  if(NOT _type STREQUAL "EXECUTABLE" AND NOT _type STREQUAL "MODULE_LIBRARY" AND NOT _type STREQUAL "SHARED_LIBRARY")
    list(REMOVE_ITEM _targets "${_target}")
  endif()
endforeach()

if(NOT CMAKE_HOST_WIN32)
  # Asan tests dont work through Wine
  if(TARGET stacksmash-asan)
    set_property(TEST stacksmash-asan PROPERTY DISABLED true)
  endif()
  if(TARGET stacksmash-asan-cfguard)
    set_property(TEST stacksmash-asan-cfguard PROPERTY DISABLED true)
  endif()
endif()

include(GNUInstallDirs)
install(TARGETS ${_targets}
        BUNDLE DESTINATION .
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)
include(CTest)
