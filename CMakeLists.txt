cmake_minimum_required(VERSION 3.13)

project(
  Stx
  VERSION 0.0.1
  DESCRIPTION "A set of utility extensions to the C++ Standard Library"
  LANGUAGES CXX)

set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake-modules)

include(CMakeDependentOption)
include(CheckIncludeFileCXX)
include(HandleFlags)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# ===============================================
#
# === Configuration Options
#
# ===============================================

option(STX_BUILD_SHARED "Build STX as a shared library" OFF)

option(STX_BUILD_TESTS "Build tests" OFF)

option(STX_ENABLE_DEBUG_ASSERTIONS "Enable debug mode assertions"
       ${STX_BUILD_TESTS})

option(STX_BUILD_DOCS "Build documentation" OFF)

option(STX_BUILD_BENCHMARKS "Build benchmarks" OFF)

cmake_dependent_option(STX_SANITIZE_TESTS "Build sanitized tests" ON
                       "STX_BUILD_TESTS" OFF)

option(
  STX_OVERRIDE_PANIC_HANDLER
  "Override the default panic behaviour (You can read the docs for more details)"
  OFF) # this feature replaces the default panic handler, by default it is set
       # to print the panic report and abort the program. The handler is re-
       # entrant and thread-safe.

option(STX_VISIBLE_PANIC_HOOK "Make runtime panic hook visible"
       ${STX_BUILD_SHARED}) # the default panic hook calls the
# provided panic handler. the behaviour can be changed at runtime especially
# when built as a shared library. Making panic hooks visible enables you to
# control the panic behaviour of the dynamically loaded library that depends on
# STX, i.e. for device drivers you might not want them to cause the whole
# program to fail on a panic.

option(STX_ENABLE_BACKTRACE "Enables the backtrace feature" OFF) # enables the
# backtrace library, we won't allow it to work by default when built as a shared
# library for now until we find a reason to, If you find it inconvenient please
# drop a PR!.

cmake_dependent_option(
  STX_ENABLE_PANIC_BACKTRACE "Enables the panic backtrace feature" ON
  "STX_ENABLE_BACKTRACE;NOT STX_OVERRIDE_PANIC_HANDLER" OFF) # disables
# printing backtraces on panic by the default panic handler. This requires that
# panic backtraces are enabled and that the panic handler is not overriden.

option(STX_USE_LIBCPP "Use Clang's libc++" OFF)

# ===============================================
#
# === Configuration Options Logging
#
# ===============================================

message(STATUS "[STX] Build as a shared library: " ${STX_BUILD_SHARED})
message(STATUS "[STX] Build tests: " ${STX_BUILD_TESTS})
message(STATUS "[STX] Enable debug mode assertions: "
               ${STX_ENABLE_DEBUG_ASSERTIONS})
message(STATUS "[STX] Build documentation: " ${STX_BUILD_DOCS})
message(STATUS "[STX] Build benchmarks: " ${STX_BUILD_BENCHMARKS})
message(STATUS "[STX] Build sanitized tests: " ${STX_SANITIZE_TESTS})
message(STATUS "[STX] Override panic handler: " ${STX_OVERRIDE_PANIC_HANDLER})
message(STATUS "[STX] Make runtime panic hook visible: "
               ${STX_VISIBLE_PANIC_HOOK})
message(STATUS "[STX] Enable backtrace: " ${STX_ENABLE_BACKTRACE})
message(STATUS "[STX] Enable panic backtrace: " ${STX_ENABLE_PANIC_BACKTRACE})
message(STATUS "[STX] Use clang's libc++: " ${STX_USE_LIBCPP})

# ===============================================
#
# === Flags and Global Definitions
#
# ===============================================
list(APPEND STX_COMPILER_DEFS)
list(APPEND STX_FLAGS)
list(APPEND STX_WARNING_FLAGS)
list(APPEND STX_TEST_WARNING_FLAGS) # tests make use of macros which don't use
                                    # the result, especially EXPECT_ANY_THROW,
                                    # the result is thus discarded and
                                    # [[nodiscard]] will raise warnings
# ===============================================
#
# === Configure Flags
#
# ===============================================

if(STX_USE_LIBCPP) # equivalent for MSVC?
  append_flags_if_supported(STX_FLAGS "-stdlib=libc++")
endif()

if(MSVC)
  append_flags_if_supported(STX_WARNING_FLAGS "/W3" "/WX")
else()
  append_flags_if_supported(STX_WARNING_FLAGS "-Wpedantic" "-Wall" "-Wextra")
endif()

append_flags_if_supported(
  STX_TEST_WARNING_FLAGS "-Wno-unused-result" "-Wno-unused-variable"
)# if available, add MSVC equivalent

# ===============================================
#
# === Configuration Definitions and Platform Support
#
# ===============================================

if(STX_ENABLE_DEBUG_ASSERTIONS)
  list(APPEND STX_COMPILER_DEFS "STX_ENABLE_DEBUG_ASSERTIONS")
endif()

if(STX_OVERRIDE_PANIC_HANDLER)
  list(APPEND STX_COMPILER_DEFS "STX_OVERRIDE_PANIC_HANDLER")
endif()

if(STX_VISIBLE_PANIC_HOOK)
  list(APPEND STX_COMPILER_DEFS "STX_VISIBLE_PANIC_HOOK")
endif()

if(STX_ENABLE_PANIC_BACKTRACE)
  list(APPEND STX_COMPILER_DEFS "STX_ENABLE_PANIC_BACKTRACE")
endif()

if(STX_ENABLE_BACKTRACE)
  # check platform support
endif()

# ===============================================
#
# === Dependencies
#
# ===============================================

if(STX_ENABLE_BACKTRACE)
  if(NOT EXISTS third_party/abseil)
    execute_process(
      COMMAND git submodule update --init --recursive third_party/abseil
      WORKING_DIRECTORY ${CMAKE_CURRENT_LIST_DIR})
  endif()
  add_subdirectory(third_party/abseil)
endif()

if(ANDROID)
  add_link_flags_if_supported(-latomic)
endif()

# ===============================================
#
# === Source Files
#
# ===============================================

if(STX_ENABLE_BACKTRACE)
  list(APPEND STX_SRCS src/backtrace.cc)
endif()

list(APPEND STX_SRCS src/panic/hook.cc src/panic.cc)

# ===============================================
#
# === Library Setup
#
# ===============================================

if(STX_BUILD_SHARED)
  add_library(stx SHARED ${STX_SRCS})
else()
  add_library(stx STATIC ${STX_SRCS})
endif()

target_include_directories(stx PUBLIC include)
set_target_properties(stx PROPERTIES CXX_STANDARD 20 CXX_STANDARD_REQUIRED ON)
target_compile_options(stx PRIVATE ${STX_FLAGS} ${STX_WARNING_FLAGS})
target_compile_definitions(stx PUBLIC ${STX_COMPILER_DEFS})

if(STX_ENABLE_BACKTRACE)
  target_link_libraries(stx absl::stacktrace absl::symbolize)
endif()

# ===============================================
#
# === Test Dependencies
#
# ===============================================

if(STX_BUILD_TESTS OR STX_BUILD_BENCHMARKS) # benchmark also uses it

  if(NOT EXISTS third_party/fmt)
    execute_process(
      COMMAND git submodule update --init --recursive third_party/fmt
      WORKING_DIRECTORY ${CMAKE_CURRENT_LIST_DIR})
  endif()

  set(FMT_TEST
      OFF
      CACHE BOOL "Build FMT tests" FORCE)
  add_subdirectory(third_party/fmt)
endif()

if(STX_BUILD_TESTS)

  if(NOT EXISTS third_party/googletest)
    execute_process(
      COMMAND git submodule update --init --recursive third_party/googletest
      WORKING_DIRECTORY ${CMAKE_CURRENT_LIST_DIR})
  endif()

  set(BUILD_GMOCK
      OFF
      CACHE BOOL "Build GMOCK" FORCE)
  set(INSTALL_GTEST
      OFF
      CACHE BOOL "Install GTest" FORCE)
  add_subdirectory(third_party/googletest)
endif()

# ===============================================
#
# === Benchmark Dependencies
#
# ===============================================

if(STX_BUILD_BENCHMARKS)

  if(NOT EXISTS third_party/benchmark)
    execute_process(
      COMMAND git submodule update --init --recursive third_party/benchmark
      WORKING_DIRECTORY ${CMAKE_CURRENT_LIST_DIR})
  endif()

  set(BENCHMARK_ENABLE_TESTING OFF)
  add_subdirectory(third_party/benchmark)
endif()

# ===============================================
#
# === Documentation Dependencies
#
# ===============================================

if(STX_BUILD_DOCS)
  find_package(Doxygen) # sub-packages?
  if(NOT Doxygen_FOUND)
    message(
      WARNING "Doxygen can not be found, Documentation will not be built.")
    set(STX_BUILD_DOCS
        OFF
        CACHE BOOL "Build documentation" FORCE)
  endif()
endif()

# ===============================================
#
# === Tests Setup
#
# ===============================================

list(
  APPEND STX_TEST_SRCS
         tests/tests.cc
         tests/panic_test.cc
         tests/result_test.cc
         tests/common_test.cc
         tests/option_test.cc
         tests/report_test.cc)

if(STX_ENABLE_BACKTRACE)
  list(APPEND STX_TEST_SRCS tests/backtrace_test.cc)
endif()

if(STX_BUILD_TESTS)

  add_executable(stx_tests ${STX_TEST_SRCS})
  target_link_libraries(stx_tests stx gtest_main gtest)
  target_include_directories(stx_tests PRIVATE tests)
  target_compile_options(stx_tests PRIVATE ${STX_WARNING_FLAGS}
                                           ${STX_TEST_WARNING_FLAGS})
  target_compile_definitions(stx_tests PRIVATE STX_OVERRIDE_PANIC_HANDLER)
  set_target_properties(stx_tests
                        PROPERTIES CXX_STANDARD 20 CXX_STANDARD_REQUIRED ON)
  add_custom_target(
    run_tests
    COMMAND ${CMAKE_CURRENT_BINARY_DIR}/stx_tests
    DEPENDS stx_tests)

  macro(add_sanitized_test sanitize_mode)
    if(${CMAKE_CXX_COMPILER_ID} MATCHES "GNU|Clang") # only known to be
                                                     # supported on GCC and
                                                     # Clang
      add_executable("stx_tests_${sanitize_mode}_sanitized" ${STX_TEST_SRCS})
      target_compile_options(
        "stx_tests_${sanitize_mode}_sanitized"
        PRIVATE "-fsanitize=${sanitize_mode}" ${STX_WARNING_FLAGS}
                ${STX_TEST_WARNING_FLAGS})
      target_compile_definitions("stx_tests_${sanitize_mode}_sanitized"
                                 PRIVATE STX_OVERRIDE_PANIC_HANDLER)
      target_include_directories("stx_tests_${sanitize_mode}_sanitized"
                                 PRIVATE tests)
      target_link_libraries("stx_tests_${sanitize_mode}_sanitized" stx
                            gtest_main gtest "-fsanitize=${sanitize_mode}")
      set_target_properties("stx_tests_${sanitize_mode}_sanitized"
                            PROPERTIES CXX_STANDARD 20 CXX_STANDARD_REQUIRED ON)
    endif()
  endmacro(add_sanitized_test sanitize_mode)

  if(STX_SANITIZE_TESTS)
    add_sanitized_test(undefined)
    add_sanitized_test(address)
    add_sanitized_test(leak)
    add_sanitized_test(thread)
  endif()

endif()

# ===============================================
#
# === Benchmark Setup
#
# ===============================================

if(STX_BUILD_BENCHMARKS)

  macro(add_benchmark benchmark_name file)
    add_executable("stx_benchmark_${benchmark_name}" "benchmarks/${file}")
    target_link_libraries("stx_benchmark_${benchmark_name}" stx benchmark
                          benchmark_main)
    set_target_properties("stx_benchmark_${benchmark_name}"
                          PROPERTIES CXX_STANDARD 20 CXX_STANDARD_REQUIRED ON)
    target_compile_options("stx_benchmark_${benchmark_name}"
                           PRIVATE ${STX_WARNING_FLAGS})
  endmacro(add_benchmark benchmark_name)

  add_benchmark(one_op one_op.cc)
  add_benchmark(two_op two_op.cc)

endif()

# ===============================================
#
# === Documentation Setup
#
# ===============================================

if(STX_BUILD_DOCS)
  set(STX_DOXYGEN_IN ${CMAKE_CURRENT_LIST_DIR}/Doxyfile.in)
  set(STX_DOXYGEN_OUT ${CMAKE_CURRENT_BINARY_DIR}/docs/Doxyfile)

  # request to configure the file
  configure_file(${STX_DOXYGEN_IN} ${STX_DOXYGEN_OUT} @ONLY)
  message(STATUS "Doxygen build started")

  # note the option ALL which allows to build the docs together with the
  # application
  add_custom_target(
    docs ALL
    COMMAND ${DOXYGEN_EXECUTABLE} ${STX_DOXYGEN_OUT}
    WORKING_DIRECTORY ${CMAKE_CURRENT_LIST_DIR}
    COMMENT "Generating API documentation with Doxygen"
    VERBATIM)

  file(COPY assets/fonts assets/images
       DESTINATION ${CMAKE_CURRENT_BINARY_DIR}/docs/html/assets)

endif()
