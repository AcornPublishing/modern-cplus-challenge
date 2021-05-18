cmake_minimum_required (VERSION 3.7.0) 
project(cryptlib)

# Need to set SRC_DIR manually after removing the Python library code.
set(SRC_DIR ${CMAKE_CURRENT_SOURCE_DIR})

# Make RelWithDebInfo the default (it does e.g. add '-O2 -g -DNDEBUG' for GNU)
#   If not in multi-configuration environments, no explicit build type or CXX
#   flags are set by the user and if we are the root CMakeLists.txt file.
if (NOT CMAKE_CONFIGURATION_TYPES AND
    NOT CMAKE_NO_BUILD_TYPE AND
    NOT CMAKE_BUILD_TYPE AND
    NOT CMAKE_CXX_FLAGS AND
    CMAKE_SOURCE_DIR STREQUAL CMAKE_CURRENT_SOURCE_DIR)
  set(CMAKE_BUILD_TYPE RelWithDebInfo)
endif ()

include(GNUInstallDirs)
include(CheckCXXCompilerFlag)

# We now carry around test programs. test_cxx.cxx is the default C++ one.
# Also see https://github.com/weidai11/cryptopp/issues/741.

#============================================================================
# Settable options
#============================================================================

# These are IA-32 options. TODO: Add ARM A-32, Aarch64 and Power8 options.
option(DISABLE_ASM "Disable ASM" OFF)
option(DISABLE_SSSE3 "Disable SSSE3" OFF)
option(DISABLE_SSE4 "Disable SSE4" OFF)
option(DISABLE_AESNI "Disable AES-NI" OFF)
option(DISABLE_SHA "Disable SHA" OFF)
option(DISABLE_AVX "Disable AVX" OFF)
option(DISABLE_AVX2 "Disable AVX2" OFF)
option(CRYPTOPP_NATIVE_ARCH "Enable native architecture" OFF)
set(CRYPTOPP_DATA_DIR "" CACHE PATH "Crypto++ test data directory")

#============================================================================
# Compiler options
#============================================================================

set(CRYPTOPP_COMPILE_DEFINITIONS)
set(CRYPTOPP_COMPILE_OPTIONS)

# Stop hiding the damn output...
# set(CMAKE_VERBOSE_MAKEFILE on)

# Always 1 ahead in Master. Also see http://groups.google.com/forum/#!topic/cryptopp-users/SFhqLDTQPG4
set(LIB_VER ${cryptopp_VERSION_MAJOR}${cryptopp_VERSION_MINOR}${cryptopp_VERSION_PATCH})

# Don't use RPATH's. The resulting binary could fail a security audit.
set(CMAKE_MACOSX_RPATH 0)

if (CMAKE_CXX_COMPILER_ID STREQUAL "Intel")
  list(APPEND CRYPTOPP_COMPILE_OPTIONS -wd68 -wd186 -wd279 -wd327 -wd161 -wd3180)
endif ()

# Also see http://github.com/weidai11/cryptopp/issues/395
if (DISABLE_ASM)
  list(APPEND CRYPTOPP_COMPILE_DEFINITIONS CRYPTOPP_DISABLE_ASM)
endif ()
if (DISABLE_SSSE3)
  list(APPEND CRYPTOPP_COMPILE_DEFINITIONS CRYPTOPP_DISABLE_SSSE3)
endif ()
if (DISABLE_SSE4)
  list(APPEND CRYPTOPP_COMPILE_DEFINITIONS CRYPTOPP_DISABLE_SSSE4)
endif ()
if (DISABLE_AESNI)
  list(APPEND CRYPTOPP_COMPILE_DEFINITIONS CRYPTOPP_DISABLE_AESNI)
endif ()
if (DISABLE_SHA)
  list(APPEND CRYPTOPP_COMPILE_DEFINITIONS CRYPTOPP_DISABLE_SHA)
endif ()
if (DISABLE_ALTIVEC)
  list(APPEND CRYPTOPP_COMPILE_DEFINITIONS CRYPTOPP_DISABLE_ALTIVEC)
endif ()
if (DISABLE_POWER7)
  list(APPEND CRYPTOPP_COMPILE_DEFINITIONS CRYPTOPP_DISABLE_POWER7)
endif ()
if (DISABLE_POWER8)
  list(APPEND CRYPTOPP_COMPILE_DEFINITIONS CRYPTOPP_DISABLE_POWER8)
endif ()
if (DISABLE_POWER9)
  list(APPEND CRYPTOPP_COMPILE_DEFINITIONS CRYPTOPP_DISABLE_POWER9)
endif ()
if (NOT CRYPTOPP_DATA_DIR STREQUAL "")
  list(APPEND CRYPTOPP_COMPILE_DEFINITIONS "CRYPTOPP_DATA_DIR=${CRYPTOPP_DATA_DIR}")
endif ()

###############################################################################

# Try to find a Posix compatible grep and sed. Solaris, Digital Unix,
#   Tru64, HP-UX and a few others need tweaking

if (EXISTS /usr/xpg4/bin/grep)
  set(GREP_CMD /usr/xpg4/bin/grep)
elseif (EXISTS /usr/gnu/bin/grep)
  set(GREP_CMD /usr/gnu/bin/grep)
elseif (EXISTS /usr/linux/bin/grep)
  set(GREP_CMD /usr/linux/bin/grep)
else ()
  set(GREP_CMD grep)
endif ()

if (EXISTS /usr/xpg4/bin/sed)
  set(SED_CMD /usr/xpg4/bin/sed)
elseif (EXISTS /usr/gnu/bin/sed)
  set(SED_CMD /usr/gnu/bin/sed)
elseif (EXISTS /usr/linux/bin/sed)
  set(SED_CMD /usr/linux/bin/sed)
else ()
  set(SED_CMD sed)
endif ()

###############################################################################

function(CheckCompileOption opt var)
  CHECK_CXX_COMPILER_FLAG(${opt} ${var})
endfunction(CheckCompileOption)

function(CheckCompileLinkOption opt var prog)

  if (MSVC)

    # TODO: improve this...
    CHECK_CXX_COMPILER_FLAG(${opt} ${var})

  else ()

    message(STATUS "Performing Test ${var}")
    execute_process(
      COMMAND sh -c "${CMAKE_CXX_COMPILER} ${CMAKE_CXX_FLAGS} ${opt} ${prog} 2>&1"
      RESULT_VARIABLE COMMAND_RESULT
      OUTPUT_VARIABLE COMMAND_OUTPUT
      OUTPUT_STRIP_TRAILING_WHITESPACE)

    # message(STATUS "RESULT_VARIABLE ${RESULT_VARIABLE}")
    # message(STATUS "COMMAND_RESULT ${COMMAND_RESULT}")
    # message(STATUS "OUTPUT_VARIABLE ${OUTPUT_VARIABLE}")
    # message(STATUS "COMMAND_OUTPUT ${COMMAND_OUTPUT}")

    # This test is strict. We require two things. First, the invocation
    # of the compile command must return 0. Second, there must be no
    # messages on the console. We are interested in diagnostics like
    # warnings to decide when to reject an option. But we will probably
    # capture chatty compiler that want to say, "Hooray, success". For
    # chatty compilers we will need to find a quiet option and use it
    # for the test. Microsoft compilers come to mind.
    if ("${COMMAND_RESULT}" EQUAL 0 AND "${COMMAND_OUTPUT}" STREQUAL "")
      set(${var} 1 PARENT_SCOPE)
      message(STATUS "Performing Test ${var} - Success")
    else ()
      set(${var} 0 PARENT_SCOPE)
      message(STATUS "Performing Test ${var} - Failed")
    endif ()

  endif ()

endfunction(CheckCompileLinkOption)

function(AddCompileOption opt)

    if ("${COMMAND_OUTPUT}" NOT STREQUAL "")
      list(APPEND CRYPTOPP_COMPILE_OPTIONS "${opt}")
    endif ()

endfunction(AddCompileOption)

###############################################################################

function(DumpMachine output pattern)

  if (MSVC)

    # CMake does not provide a generic shell/terminal mechanism
    #  and Microsoft environments don't know what 'sh' is.
    set(${output} 0 PARENT_SCOPE)

  else ()

    execute_process(
      COMMAND sh -c "${CMAKE_CXX_COMPILER} -dumpmachine 2>&1"
      COMMAND ${GREP_CMD} -i -c -E "${pattern}"
      OUTPUT_VARIABLE ${output}
      OUTPUT_STRIP_TRAILING_WHITESPACE)
    set(${output} "${${output}}" PARENT_SCOPE)

  endif()

endfunction(DumpMachine)

# Thansk to Anonimal for MinGW; see http://github.com/weidai11/cryptopp/issues/466
DumpMachine(CRYPTOPP_AMD64 "amd64|x86_64")
DumpMachine(CRYPTOPP_I386 "i.86")
DumpMachine(CRYPTOPP_X32 "x32")

###############################################################################

if (CMAKE_CXX_COMPILER_ID STREQUAL "GNU")

  execute_process(COMMAND sh -c "${CMAKE_CXX_COMPILER} --version 2>&1"
    COMMAND ${GREP_CMD} -i -c "macports"
    OUTPUT_VARIABLE MACPORTS
    OUTPUT_STRIP_TRAILING_WHITESPACE)

  if (MACPORTS EQUAL 0)
    # Get GAS version, add defs + set as appropriate
    set(GAS_CMD sh -c "${CMAKE_CXX_COMPILER} -xc -c /dev/null -Wa,-v -o/dev/null 2>&1")

    execute_process(COMMAND ${GAS_CMD}
      OUTPUT_VARIABLE GAS_STRING
      OUTPUT_STRIP_TRAILING_WHITESPACE)
    string(FIND "${GAS_STRING}" "GNU assembler" GAS_OUTPUT)

    if (NOT GAS_OUTPUT EQUAL -1)
      #.intel_syntax wasn't supported until GNU assembler 2.10

      # TODO(unassigned): string() REGEX was not cooperating at time of writing. Re-implement as needed.
      execute_process(COMMAND echo ${GAS_STRING}
        COMMAND ${GREP_CMD} -i -c -E "GNU.[Aa]ssembler.*(2\\.[1-9][0-9]|[3-9])"
        OUTPUT_VARIABLE GAS210_OR_LATER)
      if (GAS210_OR_LATER EQUAL 0)
        list(APPEND CRYPTOPP_COMPILE_DEFINITIONS CRYPTOPP_DISABLE_ASM)
        set(DISABLE_ASM 1)
      endif ()

      execute_process(COMMAND echo ${GAS_STRING}
        COMMAND ${GREP_CMD} -i -c -E "GNU.[Aa]ssembler.*(2\\.1[7-9]|2\\.[2-9]|[3-9])"
        OUTPUT_VARIABLE GAS217_OR_LATER)
      if (GAS217_OR_LATER EQUAL 0)
        list(APPEND CRYPTOPP_COMPILE_DEFINITIONS CRYPTOPP_DISABLE_SSSE3)
        set(DISABLE_SSSE3 1)
      endif ()

      # OpenBSD and CentOS 5 needed this one due to ARIA and BLAKE2
      execute_process(COMMAND echo ${GAS_STRING}
        COMMAND ${GREP_CMD} -i -c -E "GNU.[Aa]ssembler.*(2\\.1[8-9]|2\\.[2-9]|[3-9])"
        OUTPUT_VARIABLE GAS218_OR_LATER)
      if (GAS218_OR_LATER EQUAL 0)
        list(APPEND CRYPTOPP_COMPILE_DEFINITIONS CRYPTOPP_DISABLE_SSSE4)
        set(DISABLE_SSE4 1)
      endif ()

      execute_process(COMMAND echo ${GAS_STRING}
        COMMAND ${GREP_CMD} -i -c -E "GNU.[Aa]ssembler.*(2\\.19|2\\.[2-9]|[3-9])"
        OUTPUT_VARIABLE GAS219_OR_LATER)
      if (GAS219_OR_LATER EQUAL 0)
        list(APPEND CRYPTOPP_COMPILE_DEFINITIONS CRYPTOPP_DISABLE_AESNI)
        set(DISABLE_AESNI 1)
      endif ()

      # Ubuntu 10 and Ubuntu 12 needed this one
      execute_process(COMMAND echo ${GAS_STRING}
        COMMAND ${GREP_CMD} -i -c -E "GNU.[Aa]ssembler.*(2\\.2[3-9]|2\\.[3-9]|[3-9])"
        OUTPUT_VARIABLE GAS223_OR_LATER)
      if (GAS223_OR_LATER EQUAL 0)
        list(APPEND CRYPTOPP_COMPILE_DEFINITIONS CRYPTOPP_DISABLE_SHA)
        set(DISABLE_SHA 1)
      endif ()
    endif ()
  endif ()
endif ()

# TODO: what about ICC and LLVM on Windows?
if (MSVC)
  if (CMAKE_SYSTEM_VERSION MATCHES "10\\.0.*")
    list(APPEND CRYPTOPP_COMPILE_DEFINITIONS "_WIN32_WINNT=0x0A00")
  endif ()
  list(APPEND CRYPTOPP_COMPILE_OPTIONS "/FI\"winapifamily.h\"")
endif ()

# Enable PIC for all target machines except 32-bit i386 due to register pressures.
if (NOT CRYPTOPP_I386)
  SET(CMAKE_POSITION_INDEPENDENT_CODE 1)
endif ()

#============================================================================
# Sources & headers
#============================================================================

# Library headers
file(GLOB cryptopp_HEADERS ${SRC_DIR}/*.h)

# Remove headers used to build test suite
list(REMOVE_ITEM cryptopp_HEADERS
    ${SRC_DIR}/bench.h
    ${SRC_DIR}/validate.h
    )

# Library sources. You can use the GNUmakefile to generate the list: `make sources`.
# Makefile sorted them at http://github.com/weidai11/cryptopp/pull/426.
file(GLOB cryptopp_SOURCES ${SRC_DIR}/*.cpp)
list(SORT cryptopp_SOURCES)
list(REMOVE_ITEM cryptopp_SOURCES
    ${SRC_DIR}/cryptlib.cpp
    ${SRC_DIR}/cpu.cpp
    ${SRC_DIR}/integer.cpp
    ${SRC_DIR}/pch.cpp
    ${SRC_DIR}/simple.cpp
    ${SRC_DIR}/adhoc.cpp
    )
set(cryptopp_SOURCES
    ${SRC_DIR}/cryptlib.cpp
    ${SRC_DIR}/cpu.cpp
    ${SRC_DIR}/integer.cpp
    ${cryptopp_SOURCES}
    )

set(cryptopp_SOURCES_ASM)

if (MSVC AND NOT DISABLE_ASM)
  if (${CMAKE_GENERATOR} MATCHES ".*ARM")
    message(STATUS "Disabling ASM because ARM is specified as target platform.")
  else ()
    enable_language(ASM_MASM)
    list(APPEND cryptopp_SOURCES_ASM
      ${SRC_DIR}/rdrand.asm
      )
    if (CMAKE_SIZEOF_VOID_P EQUAL 8)
      list(APPEND cryptopp_SOURCES_ASM
        ${SRC_DIR}/x64dll.asm
        ${SRC_DIR}/x64masm.asm
        )
      set_source_files_properties(${cryptopp_SOURCES_ASM} PROPERTIES COMPILE_DEFINITIONS "_M_X64")
    else ()
      set_source_files_properties(${cryptopp_SOURCES_ASM} PROPERTIES COMPILE_DEFINITIONS "_M_X86" COMPILE_FLAGS "/safeseh")
    endif ()
    set_source_files_properties(${cryptopp_SOURCES_ASM} PROPERTIES LANGUAGE ASM_MASM)
  endif ()
endif ()

#============================================================================
# Architecture flags
#============================================================================

# TODO: Android, AIX, IBM xlC, iOS and a few other profiles are missing.

# New as of Pull Request 461, http://github.com/weidai11/cryptopp/pull/461.
# Must use CMAKE_CXX_COMPILER here due to XLC 13.1 and LLVM front-end.
if (CMAKE_CXX_COMPILER_ID MATCHES "Clang" OR CMAKE_CXX_COMPILER_ID STREQUAL "GNU" OR CMAKE_CXX_COMPILER_ID STREQUAL "Intel" OR CMAKE_CXX_COMPILER MATCHES "xlC")

  if (CRYPTOPP_AMD64 OR CRYPTOPP_I386 OR CRYPTOPP_X32)

    # https://github.com/weidai11/cryptopp/issues/756
    if (NOT CRYPTOPP_MIXED_ASM)
      list(APPEND CRYPTOPP_COMPILE_OPTIONS "-DCRYPTOPP_DISABLE_MIXED_ASM")
    endif ()

    if (NOT CRYPTOPP_IA32_SSE2 AND NOT DISABLE_ASM)
      list(APPEND CRYPTOPP_COMPILE_OPTIONS "-DCRYPTOPP_DISABLE_ASM")
    elseif (CRYPTOPP_IA32_SSE2 AND NOT DISABLE_ASM)
      set_source_files_properties(${SRC_DIR}/sse_simd.cpp PROPERTIES COMPILE_FLAGS "-msse2")
      set_source_files_properties(${SRC_DIR}/chacha_simd.cpp PROPERTIES COMPILE_FLAGS "-msse2")
      set_source_files_properties(${SRC_DIR}/donna_sse.cpp PROPERTIES COMPILE_FLAGS "-msse2")
    endif ()
    if (NOT CRYPTOPP_IA32_SSSE3 AND NOT DISABLE_SSSE3)
      list(APPEND CRYPTOPP_COMPILE_OPTIONS "-DCRYPTOPP_DISABLE_SSSE3")
    elseif (CRYPTOPP_IA32_SSSE3 AND NOT DISABLE_SSSE3)
      set_source_files_properties(${SRC_DIR}/aria_simd.cpp PROPERTIES COMPILE_FLAGS "-mssse3")
      set_source_files_properties(${SRC_DIR}/cham_simd.cpp PROPERTIES COMPILE_FLAGS "-mssse3")
      set_source_files_properties(${SRC_DIR}/keccak_simd.cpp PROPERTIES COMPILE_FLAGS "-mssse3")
      set_source_files_properties(${SRC_DIR}/lea_simd.cpp PROPERTIES COMPILE_FLAGS "-mssse3")
      set_source_files_properties(${SRC_DIR}/simeck_simd.cpp PROPERTIES COMPILE_FLAGS "-mssse3")
      set_source_files_properties(${SRC_DIR}/simon128_simd.cpp PROPERTIES COMPILE_FLAGS "-mssse3")
      set_source_files_properties(${SRC_DIR}/speck128_simd.cpp PROPERTIES COMPILE_FLAGS "-mssse3")
      if (NOT CRYPTOPP_IA32_SSE41 AND NOT DISABLE_SSE4)
        list(APPEND CRYPTOPP_COMPILE_OPTIONS "-DCRYPTOPP_DISABLE_SSE4")
      elseif (CRYPTOPP_IA32_SSE41 AND NOT DISABLE_SSE4)
        set_source_files_properties(${SRC_DIR}/blake2s_simd.cpp PROPERTIES COMPILE_FLAGS "-msse4.1")
        set_source_files_properties(${SRC_DIR}/blake2b_simd.cpp PROPERTIES COMPILE_FLAGS "-msse4.1")
        set_source_files_properties(${SRC_DIR}/simon64_simd.cpp PROPERTIES COMPILE_FLAGS "-msse4.1")
        set_source_files_properties(${SRC_DIR}/speck64_simd.cpp PROPERTIES COMPILE_FLAGS "-msse4.1")
      endif ()
      if (NOT CRYPTOPP_IA32_SSE42 AND NOT DISABLE_SSE4)
        list(APPEND CRYPTOPP_COMPILE_OPTIONS "-DCRYPTOPP_DISABLE_SSE4")
      elseif (CRYPTOPP_IA32_SSE42 AND NOT DISABLE_SSE4)
        set_source_files_properties(${SRC_DIR}/crc_simd.cpp PROPERTIES COMPILE_FLAGS "-msse4.2")
        if (NOT CRYPTOPP_IA32_CLMUL AND NOT DISABLE_AES)
          list(APPEND CRYPTOPP_COMPILE_OPTIONS "-DCRYPTOPP_DISABLE_CLMUL")
        elseif (CRYPTOPP_IA32_CLMUL AND NOT DISABLE_AES)
          set_source_files_properties(${SRC_DIR}/gcm_simd.cpp PROPERTIES COMPILE_FLAGS "-mssse3 -mpclmul")
          set_source_files_properties(${SRC_DIR}/gf2n_simd.cpp PROPERTIES COMPILE_FLAGS "-mpclmul")
        endif ()
        if (NOT CRYPTOPP_IA32_AES AND NOT DISABLE_AES)
          list(APPEND CRYPTOPP_COMPILE_OPTIONS "-DCRYPTOPP_DISABLE_AESNI")
        elseif (CRYPTOPP_IA32_AES AND NOT DISABLE_AES)
          set_source_files_properties(${SRC_DIR}/rijndael_simd.cpp PROPERTIES COMPILE_FLAGS "-msse4.1 -maes")
          set_source_files_properties(${SRC_DIR}/sm4_simd.cpp PROPERTIES COMPILE_FLAGS "-mssse3 -maes")
        endif ()
        #if (NOT CRYPTOPP_IA32_AVX AND NOT DISABLE_AVX)
        # list(APPEND CRYPTOPP_COMPILE_OPTIONS "-DCRYPTOPP_DISABLE_AVX")
        #elseif (CRYPTOPP_IA32_AVX AND NOT DISABLE_AVX)
        #  set_source_files_properties(${SRC_DIR}/XXX_avx.cpp PROPERTIES COMPILE_FLAGS "-mavx")
        #endif ()
        if (NOT CRYPTOPP_IA32_AVX2 AND NOT DISABLE_AVX2)
          list(APPEND CRYPTOPP_COMPILE_OPTIONS "-DCRYPTOPP_DISABLE_AVX2")
        elseif (CRYPTOPP_IA32_AVX2 AND NOT DISABLE_AVX2)
          set_source_files_properties(${SRC_DIR}/chacha_avx.cpp PROPERTIES COMPILE_FLAGS "-mavx2")
        endif ()
        if (NOT CRYPTOPP_IA32_SHA AND NOT DISABLE_SHA)
          list(APPEND CRYPTOPP_COMPILE_OPTIONS "-DCRYPTOPP_DISABLE_SHANI")
        elseif (CRYPTOPP_IA32_SHA AND NOT DISABLE_SHA)
          set_source_files_properties(${SRC_DIR}/sha_simd.cpp PROPERTIES COMPILE_FLAGS "-msse4.2 -msha")
          set_source_files_properties(${SRC_DIR}/shacal2_simd.cpp PROPERTIES COMPILE_FLAGS "-msse4.2 -msha")
        endif ()
      endif ()
    endif ()
  endif ()
endif ()

# Attempt to determine a suitable native option
if (CRYPTOPP_NATIVE_ARCH)

    CheckCompileOption("-march=native" NATIVE_ARCH)
    if (NATIVE_ARCH)
      list(APPEND CRYPTOPP_COMPILE_OPTIONS "-march=native")
    else ()
      CheckCompileOption("-native" NATIVE_ARCH)
      if (NATIVE_ARCH)
        list(APPEND CRYPTOPP_COMPILE_OPTIONS "-native")
      endif ()
    endif ()

    if (NOT NATIVE_ARCH)
      message(WARNING "CRYPTOPP_NATIVE_ARCH enabled, but failed to detect native architecture option")
    endif ()

endif()

#============================================================================
# Compile targets
#============================================================================

# Work around the archaic versions of cmake that do not support
# target_compile_xxxx commands
# !!! DO NOT try to use the old way for newer version - it does not work !!!

function(cryptopp_target_compile_properties target)
  target_compile_definitions(${target} PUBLIC ${CRYPTOPP_COMPILE_DEFINITIONS})
  target_compile_options(${target} PUBLIC ${CRYPTOPP_COMPILE_OPTIONS})
endfunction()

set(cryptopp_LIBRARY_SOURCES ${cryptopp_SOURCES_ASM})
list(APPEND cryptopp_LIBRARY_SOURCES
  ${cryptopp_SOURCES}
  )

add_library(cryptlib STATIC ${cryptopp_LIBRARY_SOURCES})
cryptopp_target_compile_properties(cryptlib)
target_include_directories(cryptlib PUBLIC $<BUILD_INTERFACE:${SRC_DIR}> $<INSTALL_INTERFACE:include>)

# if (NOT MSVC)
#   set_target_properties(cryptlib
#       PROPERTIES
#       OUTPUT_NAME cryptopp)
# endif ()

# Add alternate ways to invoke the build for the shared library that are
# similar to how the crypto++ 'make' tool works.
# see https://github.com/noloader/cryptopp-cmake/issues/32
add_custom_target(static DEPENDS cryptlib)
#============================================================================
# Third-party libraries
#============================================================================

if (WIN32)
  target_link_libraries(cryptlib ws2_32)
endif ()

find_package(Threads)
target_link_libraries(cryptlib ${CMAKE_THREAD_LIBS_INIT})

set_target_properties (cryptlib PROPERTIES FOLDER "00.Libs")
