cmake_minimum_required(VERSION 2.8.12...3.28)

if(CMAKE_EXECUTABLE_SUFFIX)
  set(CMAKE_EXECUTABLE_SUFFIX_TMP ${CMAKE_EXECUTABLE_SUFFIX})
endif()

project(libjpeg-turbo C)
set(VERSION 3.1.4)
set(COPYRIGHT_YEAR "1991-2024")

set(USE_OWN_NASM_INVOCATION 0)
message(STATUS "Using CMake's NASM integration")

set(VERSION_MAJOR 3)
set(VERSION_MINOR 1)
set(VERSION_REVISION 4)
function(pad_number NUMBER OUTPUT_LEN)
  string(LENGTH "${${NUMBER}}" INPUT_LEN)
  if(INPUT_LEN LESS OUTPUT_LEN)
    math(EXPR ZEROES "${OUTPUT_LEN} - ${INPUT_LEN} - 1")
    set(NUM ${${NUMBER}})
    foreach(C RANGE ${ZEROES})
      set(NUM "0${NUM}")
    endforeach()
    set(${NUMBER} ${NUM} PARENT_SCOPE)
  endif()
endfunction()
pad_number(VERSION_MINOR 3)
pad_number(VERSION_REVISION 3)
set(LIBJPEG_TURBO_VERSION_NUMBER ${VERSION_MAJOR}${VERSION_MINOR}${VERSION_REVISION})

string(TIMESTAMP DEFAULT_BUILD "%Y%m%d")
set(BUILD ${DEFAULT_BUILD} CACHE STRING "Build string (default: ${DEFAULT_BUILD})")

message(STATUS "CMAKE_BUILD_TYPE = ${CMAKE_BUILD_TYPE}")
message(STATUS "VERSION = ${VERSION}, BUILD = ${BUILD}")

include(cmakescripts/PackageInfo.cmake)

# Detect CPU type and whether we're building 64-bit or 32-bit code
math(EXPR BITS "${CMAKE_SIZEOF_VOID_P} * 8")
string(TOLOWER ${CMAKE_SYSTEM_PROCESSOR} CMAKE_SYSTEM_PROCESSOR_LC)
if(CMAKE_SYSTEM_PROCESSOR_LC MATCHES "x86_64" OR
  CMAKE_SYSTEM_PROCESSOR_LC MATCHES "amd64" OR
  CMAKE_SYSTEM_PROCESSOR_LC MATCHES "i[0-9]86" OR
  CMAKE_SYSTEM_PROCESSOR_LC MATCHES "x86" OR
  CMAKE_SYSTEM_PROCESSOR_LC MATCHES "ia32")
  if(BITS EQUAL 64 OR CMAKE_C_COMPILER_ABI MATCHES "ELF X32")
    set(CPU_TYPE x86_64)
  else()
    set(CPU_TYPE i386)
  endif()
  if(NOT CMAKE_SYSTEM_PROCESSOR STREQUAL ${CPU_TYPE})
    set(CMAKE_SYSTEM_PROCESSOR ${CPU_TYPE})
  endif()
elseif(CMAKE_SYSTEM_PROCESSOR_LC STREQUAL "aarch64" OR
  CMAKE_SYSTEM_PROCESSOR_LC MATCHES "^arm")
  if(BITS EQUAL 64)
    set(CPU_TYPE arm64)
  else()
    set(CPU_TYPE arm)
  endif()
elseif(CMAKE_SYSTEM_PROCESSOR_LC MATCHES "^ppc" OR
  CMAKE_SYSTEM_PROCESSOR_LC MATCHES "^powerpc")
  set(CPU_TYPE powerpc)
else()
  set(CPU_TYPE ${CMAKE_SYSTEM_PROCESSOR_LC})
endif()

message(STATUS "${BITS}-bit build (${CPU_TYPE})")


###############################################################################
# INSTALL DIRECTORIES
###############################################################################

include(GNUInstallDirs)

if(NOT CMAKE_INSTALL_RPATH AND CMAKE_INSTALL_PREFIX)
  set(CMAKE_INSTALL_RPATH ${CMAKE_INSTALL_FULL_LIBDIR})
  message(STATUS "Setting install rpath to ${CMAKE_INSTALL_RPATH}")
endif()


###############################################################################
# CONFIGURATION OPTIONS
###############################################################################

macro(boolean_number var)
  if(${var})
    set(${var} 1 ${ARGN})
  else()
    set(${var} 0 ${ARGN})
  endif()
endmacro()

option(REQUIRE_SIMD
  "Generate a fatal error if SIMD extensions are not available for this platform (default is to fall back to a non-SIMD build)"
  FALSE)
boolean_number(REQUIRE_SIMD)
option(WITH_ARITH_DEC
  "Include arithmetic decoding support when emulating the libjpeg v6b API/ABI"
  TRUE)
boolean_number(WITH_ARITH_DEC)
option(WITH_ARITH_ENC
  "Include arithmetic encoding support when emulating the libjpeg v6b API/ABI"
  TRUE)
boolean_number(WITH_ARITH_ENC)
if(CMAKE_C_COMPILER_ABI MATCHES "ELF X32")
  set(WITH_JAVA 0)
else()
  option(WITH_JAVA
    "Build Java wrapper for the TurboJPEG API library"
    FALSE)
  boolean_number(WITH_JAVA)
endif()
option(WITH_JPEG7
  "Emulate libjpeg v7 API/ABI (this makes TurboJpeg backward-incompatible with libjpeg v6b)"
  FALSE)
boolean_number(WITH_JPEG7)
option(WITH_JPEG8
  "Emulate libjpeg v8 API/ABI (this makes TurboJpeg backward-incompatible with libjpeg v6b)"
  FALSE)
boolean_number(WITH_JPEG8)
option(WITH_SIMD "Include SIMD extensions, if available for this platform" TRUE)
boolean_number(WITH_SIMD)
option(WITH_TURBOJPEG
  "Include the TurboJPEG API library and associated test programs" TRUE)
boolean_number(WITH_TURBOJPEG)
option(WITH_FUZZ "Build fuzz targets" FALSE)

macro(report_option var desc)
  if(${var})
    message(STATUS "${desc} enabled (${var} = ${${var}})")
  else()
    message(STATUS "${desc} disabled (${var} = ${${var}})")
  endif()
endmacro()

# Explicitly setting CMAKE_POSITION_INDEPENDENT_CODE=FALSE disables PIC for all
# targets, which will cause the shared library builds to fail.  Thus, if shared
# libraries are enabled and CMAKE_POSITION_INDEPENDENT_CODE is explicitly set
# to FALSE, we need to unset it, thus restoring the default behavior
# (automatically using PIC for shared library targets.)
if(DEFINED CMAKE_POSITION_INDEPENDENT_CODE AND
  NOT CMAKE_POSITION_INDEPENDENT_CODE AND BUILD_SHARED_LIBS)
  unset(CMAKE_POSITION_INDEPENDENT_CODE CACHE)
endif()

if(WITH_JPEG8 OR WITH_JPEG7)
  set(WITH_ARITH_ENC 1)
  set(WITH_ARITH_DEC 1)
endif()

if(WITH_ARITH_DEC)
  set(D_ARITH_CODING_SUPPORTED 1)
endif()
report_option(WITH_ARITH_DEC "Arithmetic decoding support")

if(WITH_ARITH_ENC)
  set(C_ARITH_CODING_SUPPORTED 1)
endif()
report_option(WITH_ARITH_ENC "Arithmetic encoding support")

report_option(WITH_TURBOJPEG "TurboJPEG API library")
report_option(WITH_JAVA "TurboJPEG Java wrapper")

if(NOT WITH_JPEG8)
  set(MEM_SRCDST_FUNCTIONS "global:  jpeg_mem_dest;  jpeg_mem_src;")
endif()

# 0: Original libjpeg v6b/v7/v8 API/ABI
#
# libjpeg v6b/v7 API/ABI emulation:
# 1: + In-memory source/destination managers (libjpeg-turbo 1.3.x)
# 2: + Partial image decompression functions (libjpeg-turbo 1.5.x)
# 3: + ICC functions (libjpeg-turbo 2.0.x)
# 4: + 12-bit-per-component and lossless functions (libjpeg-turbo 2.2.x)
#
# libjpeg v8 API/ABI emulation:
# 1: + Partial image decompression functions (libjpeg-turbo 1.5.x)
# 2: + ICC functions (libjpeg-turbo 2.0.x)
# 3: + 12-bit-per-component and lossless functions (libjpeg-turbo 2.2.x)
set(SO_AGE 3)
if(NOT WITH_JPEG8)
  set(SO_AGE 4)
endif()

if(WITH_JPEG8)
  set(JPEG_LIB_VERSION 80)
elseif(WITH_JPEG7)
  set(JPEG_LIB_VERSION 70)
else()
  set(JPEG_LIB_VERSION 62)
endif()

math(EXPR JPEG_LIB_VERSION_DIV10 "${JPEG_LIB_VERSION} / 10")
math(EXPR JPEG_LIB_VERSION_MOD10 "${JPEG_LIB_VERSION} % 10")
if(JPEG_LIB_VERSION STREQUAL "62")
  set(DEFAULT_SO_MAJOR_VERSION ${JPEG_LIB_VERSION})
else()
  set(DEFAULT_SO_MAJOR_VERSION ${JPEG_LIB_VERSION_DIV10})
endif()
if(JPEG_LIB_VERSION STREQUAL "80")
  set(DEFAULT_SO_MINOR_VERSION 2)
else()
  set(DEFAULT_SO_MINOR_VERSION 0)
endif()

# This causes SO_MAJOR_VERSION/SO_MINOR_VERSION to reset to defaults if
# WITH_JPEG7 or WITH_JPEG8 has changed.
if((DEFINED WITH_JPEG7_INT AND NOT WITH_JPEG7 EQUAL WITH_JPEG7_INT) OR
  (DEFINED WITH_JPEG8_INT AND NOT WITH_JPEG8 EQUAL WITH_JPEG8_INT))
  set(FORCE_SO_VERSION "FORCE")
endif()
set(WITH_JPEG7_INT ${WITH_JPEG7} CACHE INTERNAL "")
set(WITH_JPEG8_INT ${WITH_JPEG8} CACHE INTERNAL "")

set(SO_MAJOR_VERSION ${DEFAULT_SO_MAJOR_VERSION} CACHE STRING
  "Major version of the libjpeg API shared library (default: ${DEFAULT_SO_MAJOR_VERSION})"
  ${FORCE_SO_VERSION})
set(SO_MINOR_VERSION ${DEFAULT_SO_MINOR_VERSION} CACHE STRING
  "Minor version of the libjpeg API shared library (default: ${DEFAULT_SO_MINOR_VERSION})"
  ${FORCE_SO_VERSION})

set(JPEG_LIB_VERSION_DECIMAL "${JPEG_LIB_VERSION_DIV10}.${JPEG_LIB_VERSION_MOD10}")
message(STATUS "Emulating libjpeg API/ABI v${JPEG_LIB_VERSION_DECIMAL} (WITH_JPEG7 = ${WITH_JPEG7}, WITH_JPEG8 = ${WITH_JPEG8})")
message(STATUS "libjpeg API shared library version = ${SO_MAJOR_VERSION}.${SO_AGE}.${SO_MINOR_VERSION}")

# Because the TurboJPEG API library uses versioned symbols and changes the
# names of functions whenever they are modified in a backward-incompatible
# manner, it is always backward-ABI-compatible with itself, so the major and
# minor SO versions don't change.  However, we increase the middle number (the
# SO "age") whenever functions are added to the API, because adding functions
# affects forward API/ABI compatibility.
set(TURBOJPEG_SO_MAJOR_VERSION 0)
# 0: TurboJPEG 1.3.x API
# 1: TurboJPEG 1.4.x API
#    The TurboJPEG 1.5.x API modified some of the function prototypes, adding
#    the const keyword in front of pointers to unmodified buffers, but that did
#    not affect forward API/ABI compatibility.
# 2: TurboJPEG 2.0.x API
#    The TurboJPEG 2.1.x API modified the behavior of the tjDecompressHeader3()
#    function so that it accepts "abbreviated table specification" (AKA
#    "tables-only") datastreams as well as JPEG images, but that did not affect
#    forward API/ABI compatibility.
# 3: TurboJPEG 3 API
set(TURBOJPEG_SO_AGE 3)
set(TURBOJPEG_SO_VERSION 0.${TURBOJPEG_SO_AGE}.0)


###############################################################################
# COMPILER SETTINGS
###############################################################################

string(TOUPPER ${CMAKE_BUILD_TYPE} CMAKE_BUILD_TYPE_UC)

if(CMAKE_COMPILER_IS_GNUCC OR CMAKE_C_COMPILER_ID MATCHES "Clang")
  # Use the maximum optimization level for release builds
  foreach(var CMAKE_C_FLAGS_RELEASE CMAKE_C_FLAGS_RELWITHDEBINFO)
    if(${var} MATCHES "-O2")
      string(REGEX REPLACE "-O2" "-O3" ${var} "${${var}}")
    endif()
  endforeach()
endif()

set(EFFECTIVE_C_FLAGS "${CMAKE_C_FLAGS} ${CMAKE_C_FLAGS_${CMAKE_BUILD_TYPE_UC}}")
message(STATUS "Compiler flags = ${EFFECTIVE_C_FLAGS}")

set(EFFECTIVE_LD_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${CMAKE_EXE_LINKER_FLAGS_${CMAKE_BUILD_TYPE_UC}}")
message(STATUS "Linker flags = ${EFFECTIVE_LD_FLAGS}")

include(CheckCSourceCompiles)
include(CheckIncludeFiles)
include(CheckTypeSize)

check_type_size("size_t" SIZE_T)
check_type_size("unsigned long" UNSIGNED_LONG)

if(SIZE_T EQUAL UNSIGNED_LONG)
  check_c_source_compiles("int main(int argc, char **argv) { unsigned long a = argc;  return __builtin_ctzl(a); }"
    HAVE_BUILTIN_CTZL)
endif()

if(UNIX)
  if(CMAKE_CROSSCOMPILING)
    set(RIGHT_SHIFT_IS_UNSIGNED 0)
  else()
    include(CheckCSourceRuns)
    check_c_source_runs("
      #include <stdio.h>
      #include <stdlib.h>
      static int is_shifting_signed (long arg) {
        long res = arg >> 4;
        if (res == -0x7F7E80CL)
          return 1; /* right shift is signed */
        /* see if unsigned-shift hack will fix it. */
        /* we can't just test exact value since it depends on width of long... */
        res |= 0xFFFFFFFFL << (32-4);
        if (res == -0x7F7E80CL)
          return 0; /* right shift is unsigned */
        printf(\"Right shift isn't acting as I expect it to.\\\\n\");
        printf(\"I fear the JPEG software will not work at all.\\\\n\\\\n\");
        return 0; /* try it with unsigned anyway */
      }
      int main (void) {
        exit(is_shifting_signed(-0x7F7E80B1L));
      }" RIGHT_SHIFT_IS_UNSIGNED)
  endif()
endif()

if(CMAKE_C_COMPILER_ID MATCHES "Clang")
  check_c_source_compiles("extern const int table[1]; const int __attribute__((visibility(\"hidden\"))) table[1] = { 0 }; int main(void) { return table[0]; }"
    HIDDEN_WORKS)
  if(HIDDEN_WORKS)
    set(HIDDEN "__attribute__((visibility(\"hidden\")))")
    message(STATUS "HIDDEN = ${HIDDEN}")
  endif()
endif()

set(INLINE_OPTIONS "__inline__;inline")
option(FORCE_INLINE "Force function inlining" TRUE)
boolean_number(FORCE_INLINE)
if(FORCE_INLINE)
  list(INSERT INLINE_OPTIONS 0 "inline __attribute__((always_inline))")
  list(INSERT INLINE_OPTIONS 0 "__inline__ __attribute__((always_inline))")
endif()
foreach(inline ${INLINE_OPTIONS})
  check_c_source_compiles("${inline} static int foo(void) { return 0; } int main(void) { return foo(); }"
    INLINE_WORKS)
  if(INLINE_WORKS)
    set(INLINE ${inline})
    break()
  endif()
endforeach()
if(NOT INLINE_WORKS)
  message(FATAL_ERROR "Could not determine how to inline functions.")
endif()
message(STATUS "INLINE = ${INLINE} (FORCE_INLINE = ${FORCE_INLINE})")

set(THREAD_LOCAL "__thread")
check_c_source_compiles("static ${THREAD_LOCAL} int i;  int main(void) { i = 0;  return i; }"
  HAVE_THREAD_LOCAL)
if(HAVE_THREAD_LOCAL)
  message(STATUS "THREAD_LOCAL = ${THREAD_LOCAL}")
else()
  message(WARNING "Thread-local storage is not available.  The TurboJPEG API library's global error handler will not be thread-safe.")
  unset(THREAD_LOCAL)
endif()

if(UNIX AND NOT APPLE)
  file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/conftest.map "VERS_1 { global: *; };")
  set(CMAKE_REQUIRED_FLAGS
    "-Wl,--version-script,${CMAKE_CURRENT_BINARY_DIR}/conftest.map")
  check_c_source_compiles("int main(void) { return 0; }" HAVE_VERSION_SCRIPT)
  set(CMAKE_REQUIRED_FLAGS)
  file(REMOVE ${CMAKE_CURRENT_BINARY_DIR}/conftest.map)
  if(HAVE_VERSION_SCRIPT)
    message(STATUS "Linker supports GNU-style version scripts")
    set(MAPFLAG "-Wl,--version-script,")
    set(TJMAPFLAG "-Wl,--version-script,")
  else()
    message(STATUS "Linker does not support GNU-style version scripts")
    if(CMAKE_SYSTEM_NAME STREQUAL "SunOS")
      # The Solaris linker doesn't like our version script for the libjpeg API
      # library, but the version script for the TurboJPEG API library should
      # still work.
      file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/conftest.map
        "VERS_1 { global: foo;  local: *; }; VERS_2 { global: foo2; } VERS_1;")
      set(CMAKE_REQUIRED_FLAGS "-Wl,-M,${CMAKE_CURRENT_BINARY_DIR}/conftest.map -shared")
      check_c_source_compiles("int foo() { return 0; } int foo2() { return 2; }"
        HAVE_MAPFILE)
      set(CMAKE_REQUIRED_FLAGS)
      file(REMOVE ${CMAKE_CURRENT_BINARY_DIR}/conftest.map)
      if(HAVE_MAPFILE)
        message(STATUS "Linker supports mapfiles")
        set(TJMAPFLAG "-Wl,-M,")
      else()
        message(STATUS "Linker does not support mapfiles")
      endif()
    endif()
  endif()
endif()

# Generate files
configure_file(src/jversion.h.in jversion.h)
if(UNIX)
  configure_file(src/libjpeg.map.in libjpeg.map)
endif()

# Include directories and compiler definitions
include_directories(${CMAKE_CURRENT_BINARY_DIR} ${CMAKE_CURRENT_SOURCE_DIR})


###############################################################################
# TARGETS
###############################################################################

if(CMAKE_EXECUTABLE_SUFFIX_TMP)
  set(CMAKE_EXECUTABLE_SUFFIX ${CMAKE_EXECUTABLE_SUFFIX_TMP})
endif()
message(STATUS "CMAKE_EXECUTABLE_SUFFIX = ${CMAKE_EXECUTABLE_SUFFIX}")

set(JPEG16_SOURCES src/jcapistd.c src/jccolor.c src/jcdiffct.c src/jclossls.c src/jcmainct.c
  src/jcprepct.c src/jcsample.c src/jdapistd.c src/jdcolor.c src/jddiffct.c src/jdlossls.c src/jdmainct.c
  src/jdpostct.c src/jdsample.c src/jutils.c)
set(JPEG12_SOURCES ${JPEG16_SOURCES} src/jccoefct.c src/jcdctmgr.c src/jdcoefct.c
 src/jddctmgr.c src/jdmerge.c src/jfdctfst.c src/jfdctint.c src/jidctflt.c src/jidctfst.c src/jidctint.c
 src/jidctred.c src/jquant1.c src/jquant2.c)
set(JPEG_SOURCES ${JPEG12_SOURCES} src/jcapimin.c src/jchuff.c src/jcicc.c src/jcinit.c
  src/jclhuff.c src/jcmarker.c src/jcmaster.c src/jcomapi.c src/jcparam.c src/jcphuff.c src/jctrans.c
  src/jdapimin.c src/jdatadst.c src/jdatasrc.c src/jdhuff.c src/jdicc.c src/jdinput.c src/jdlhuff.c
  src/jdmarker.c src/jdmaster.c src/jdphuff.c src/jdtrans.c src/jerror.c src/jfdctflt.c src/jmemmgr.c
  src/jmemnobs.c src/jpeg_nbits.c)

if(WITH_ARITH_ENC OR WITH_ARITH_DEC)
  set(JPEG_SOURCES ${JPEG_SOURCES} src/jaricom.c)
endif()

if(WITH_ARITH_ENC)
  set(JPEG_SOURCES ${JPEG_SOURCES} src/jcarith.c)
endif()

if(WITH_ARITH_DEC)
  set(JPEG_SOURCES ${JPEG_SOURCES} src/jdarith.c)
endif()

if(WITH_SIMD)
  add_subdirectory(simd)
  if(NEON_INTRINSICS)
    add_definitions(-DNEON_INTRINSICS)
  endif()
else()
  message(STATUS "SIMD extensions: None (WITH_SIMD = ${WITH_SIMD})")
endif()

# We have to generate these here, because if the build system tries and fails
# to enable the SIMD extensions, the value of WITH_SIMD will have changed.
configure_file(src/jconfig.h.in jconfig.h)
configure_file(src/jconfigint.h.in jconfigint.h)

if(WITH_SIMD)
  message(STATUS "SIMD extensions: ${CPU_TYPE} (WITH_SIMD = ${WITH_SIMD})")
  if(USE_OWN_NASM_INVOCATION)
    set_source_files_properties(${SIMD_OBJS} PROPERTIES GENERATED 1)
    message(STATUS "Using own nasm invocation method because cmake's one sucks.")
  endif()
  set(SIMD_TARGET_OBJECTS $<TARGET_OBJECTS:simd>)
endif()

if(WITH_JAVA)
  add_subdirectory(java)
endif()

if(BUILD_SHARED_LIBS)
  # Compile a separate version of these source files with 12-bit and 16-bit
  # data precision.
  add_library(jpeg12 OBJECT ${JPEG12_SOURCES})
  set_property(TARGET jpeg12 PROPERTY COMPILE_FLAGS "-DBITS_IN_JSAMPLE=12")
  set_target_properties(jpeg12 PROPERTIES POSITION_INDEPENDENT_CODE 1)
  add_library(jpeg16 OBJECT ${JPEG16_SOURCES})
  set_property(TARGET jpeg16 PROPERTY COMPILE_FLAGS "-DBITS_IN_JSAMPLE=16")
  set_target_properties(jpeg16 PROPERTIES POSITION_INDEPENDENT_CODE 1)
  add_subdirectory(sharedlib)
endif()

if(NOT BUILD_SHARED_LIBS)
  # Compile a separate version of these source files with 12-bit and 16-bit
  # data precision.
  add_library(jpeg12 OBJECT ${JPEG12_SOURCES})
  set_property(TARGET jpeg12 PROPERTY COMPILE_FLAGS
    "-DBITS_IN_JSAMPLE=12")
  add_library(jpeg16 OBJECT ${JPEG16_SOURCES})
  set_property(TARGET jpeg16 PROPERTY COMPILE_FLAGS
    "-DBITS_IN_JSAMPLE=16")
  add_library(jpeg STATIC ${JPEG_SOURCES} ${SIMD_TARGET_OBJECTS}
    ${SIMD_OBJS} $<TARGET_OBJECTS:jpeg12>
    $<TARGET_OBJECTS:jpeg16>)
endif()

if(WITH_TURBOJPEG)
  if(BUILD_SHARED_LIBS)
    set(TURBOJPEG_SOURCES ${JPEG_SOURCES} ${SIMD_TARGET_OBJECTS} ${SIMD_OBJS}
      src/turbojpeg.c src/transupp.c src/jdatadst-tj.c src/jdatasrc-tj.c src/rdbmp.c src/rdppm.c
      src/wrbmp.c src/wrppm.c $<TARGET_OBJECTS:jpeg12> $<TARGET_OBJECTS:jpeg16>)
    set(TJMAPFILE ${CMAKE_CURRENT_SOURCE_DIR}/src/turbojpeg-mapfile)
    if(WITH_JAVA)
      set(TURBOJPEG_SOURCES ${TURBOJPEG_SOURCES} turbojpeg-jni.c)
      include_directories(${JAVA_INCLUDE_PATH} ${JAVA_INCLUDE_PATH2})
      set(TJMAPFILE ${CMAKE_CURRENT_SOURCE_DIR}/turbojpeg-mapfile.jni)
    endif()
    add_library(turbojpeg12 OBJECT src/rdppm.c src/wrppm.c)
    set_property(TARGET turbojpeg12 PROPERTY COMPILE_FLAGS
      "-DBITS_IN_JSAMPLE=12 -DPPM_SUPPORTED")
    set_target_properties(turbojpeg12 PROPERTIES POSITION_INDEPENDENT_CODE 1)
    add_library(turbojpeg16 OBJECT src/rdppm.c src/wrppm.c)
    set_property(TARGET turbojpeg16 PROPERTY COMPILE_FLAGS
      "-DBITS_IN_JSAMPLE=16 -DPPM_SUPPORTED")
    set_target_properties(turbojpeg16 PROPERTIES POSITION_INDEPENDENT_CODE 1)
    add_library(turbojpeg SHARED ${TURBOJPEG_SOURCES}
      $<TARGET_OBJECTS:turbojpeg12> $<TARGET_OBJECTS:turbojpeg16>)
    set_property(TARGET turbojpeg PROPERTY COMPILE_FLAGS
      "-DBMP_SUPPORTED -DPPM_SUPPORTED")
    set_target_properties(turbojpeg PROPERTIES
      SOVERSION ${TURBOJPEG_SO_MAJOR_VERSION} VERSION ${TURBOJPEG_SO_VERSION})
    if(TJMAPFLAG)
      set_target_properties(turbojpeg PROPERTIES
        LINK_FLAGS "${TJMAPFLAG}${TJMAPFILE}")
    endif()

  endif()

  if(NOT BUILD_SHARED_LIBS)
    add_library(turbojpeg12 OBJECT src/rdppm.c src/wrppm.c)
    set_property(TARGET turbojpeg12 PROPERTY COMPILE_FLAGS
      "-DBITS_IN_JSAMPLE=12 -DPPM_SUPPORTED")
    add_library(turbojpeg16 OBJECT src/rdppm.c src/wrppm.c)
    set_property(TARGET turbojpeg16 PROPERTY COMPILE_FLAGS
      "-DBITS_IN_JSAMPLE=16 -DPPM_SUPPORTED")
    add_library(turbojpeg STATIC ${JPEG_SOURCES} ${SIMD_TARGET_OBJECTS}
      ${SIMD_OBJS} src/turbojpeg.c src/transupp.c src/jdatadst-tj.c src/jdatasrc-tj.c src/rdbmp.c
      src/rdppm.c src/wrbmp.c src/wrppm.c $<TARGET_OBJECTS:jpeg12>
      $<TARGET_OBJECTS:jpeg16> $<TARGET_OBJECTS:turbojpeg12>
      $<TARGET_OBJECTS:turbojpeg16>)
    set_property(TARGET turbojpeg PROPERTY COMPILE_FLAGS
      "-DBMP_SUPPORTED -DPPM_SUPPORTED")

  endif()

  target_link_libraries(turbojpeg jpeg)
  target_include_directories(turbojpeg INTERFACE
    $<INSTALL_INTERFACE:include>
    $<BUILD_INTERFACE:${PROJECT_BINARY_DIR}>
    $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}>
  )
endif()

set(CDJPEG_COMPILE_FLAGS
  "-DBMP_SUPPORTED -DGIF_SUPPORTED -DPPM_SUPPORTED -DTARGA_SUPPORTED ${USE_SETMODE}")

if(NOT BUILD_SHARED_LIBS AND ENABLE_TOOLS)
  # Compile a separate version of these source files with 12-bit and 16-bit
  # data precision.
  add_library(cjpeg12 OBJECT src/rdgif.c src/rdppm.c)
  set_property(TARGET cjpeg12 PROPERTY COMPILE_FLAGS
    "-DBITS_IN_JSAMPLE=12 -DGIF_SUPPORTED -DPPM_SUPPORTED")
  add_library(cjpeg16 OBJECT src/rdgif.c src/rdppm.c)
  set_property(TARGET cjpeg16 PROPERTY COMPILE_FLAGS
    "-DBITS_IN_JSAMPLE=16 -DGIF_SUPPORTED -DPPM_SUPPORTED")
  add_executable(cjpeg src/cjpeg.c src/cdjpeg.c src/rdbmp.c src/rdgif.c src/rdppm.c
    rdswitch.c rdtarga.c $<TARGET_OBJECTS:cjpeg12>
    $<TARGET_OBJECTS:cjpeg16>)
  set_property(TARGET cjpeg PROPERTY COMPILE_FLAGS
    ${CDJPEG_COMPILE_FLAGS})
  target_link_libraries(cjpeg jpeg)

  # Compile a separate version of these source files with 12-bit and 16-bit
  # data precision.
  add_library(djpeg12 OBJECT src/rdcolmap.c src/wrgif.c src/wrppm.c)
  set_property(TARGET djpeg12 PROPERTY COMPILE_FLAGS
    "-DBITS_IN_JSAMPLE=12 -DGIF_SUPPORTED -DPPM_SUPPORTED")
  add_library(djpeg16 OBJECT src/wrppm.c)
  set_property(TARGET djpeg16 PROPERTY COMPILE_FLAGS
    "-DBITS_IN_JSAMPLE=16 -DPPM_SUPPORTED")
  add_executable(djpeg src/djpeg.c src/cdjpeg.c src/rdcolmap.c src/rdswitch.c src/wrbmp.c
    src/wrgif.c src/wrppm.c src/wrtarga.c $<TARGET_OBJECTS:djpeg12>
    $<TARGET_OBJECTS:djpeg16>)
  set_property(TARGET djpeg PROPERTY COMPILE_FLAGS
    ${CDJPEG_COMPILE_FLAGS})
  target_link_libraries(djpeg jpeg)

  add_executable(jpegtran src/jpegtran.c src/cdjpeg.c src/rdswitch.c src/transupp.c)
  target_link_libraries(jpegtran jpeg)
  set_property(TARGET jpegtran PROPERTY COMPILE_FLAGS "${USE_SETMODE}")

  add_executable(example src/example.c)
  target_link_libraries(example jpeg)
endif()

###############################################################################
# INSTALLATION
###############################################################################

set(EXE ${CMAKE_EXECUTABLE_SUFFIX})

if(WITH_TURBOJPEG)
  if(BUILD_SHARED_LIBS)
    install(TARGETS turbojpeg jpeg EXPORT TurboJpegTargets
      INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
      ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR} COMPONENT lib
      LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR} COMPONENT lib
      RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR} COMPONENT bin)
  endif()
  if(NOT BUILD_SHARED_LIBS)
    # "turbojpeg" requires target "jpeg" so it should be in an export set.
    install(TARGETS turbojpeg jpeg EXPORT TurboJpegTargets ARCHIVE
      INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
      ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR} COMPONENT lib)
  endif()
  install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/src/turbojpeg.h
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/turbojpeg COMPONENT include)
endif()

install(FILES ${CMAKE_CURRENT_BINARY_DIR}/pkgscripts/libjpeg.pc
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig COMPONENT lib)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/pkgscripts/libturbojpeg.pc
DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig COMPONENT lib)
install(FILES
  ${CMAKE_CURRENT_BINARY_DIR}/pkgscripts/TurboJpegConfig.cmake
  ${CMAKE_CURRENT_BINARY_DIR}/pkgscripts/TurboJpegConfigVersion.cmake
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/TurboJpeg
  COMPONENT lib)
install(EXPORT TurboJpegTargets
  FILE "TurboJpegTargets.cmake"
  DESTINATION ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/cmake)

install(FILES ${CMAKE_CURRENT_BINARY_DIR}/jconfig.h
  ${CMAKE_CURRENT_SOURCE_DIR}/src/jerror.h ${CMAKE_CURRENT_SOURCE_DIR}/src/jmorecfg.h
  ${CMAKE_CURRENT_SOURCE_DIR}/src/jpeglib.h
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/turbojpeg COMPONENT include)

configure_file(release/libjpeg.pc.in pkgscripts/libjpeg.pc @ONLY)
configure_file(release/libturbojpeg.pc.in pkgscripts/libturbojpeg.pc @ONLY)

include(CMakePackageConfigHelpers)
write_basic_package_version_file(
  pkgscripts/TurboJpegConfigVersion.cmake
  VERSION ${VERSION} COMPATIBILITY AnyNewerVersion)

configure_package_config_file(release/Config.cmake.in
  pkgscripts/TurboJpegConfig.cmake
  INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${CMAKE_PROJECT_NAME})
