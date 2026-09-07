# openQ4-GameLibs - CMake build
#
# A second build definition alongside meson.build, producing the same two game
# modules from the same sources. meson remains the canonical build: it is what
# CI, packaging and the engine's staged build use, and this file is expected to
# follow it rather than lead it.
#
# Both builds share the repository's own helper scripts - src/buildscripts/
# list_sources.py for source discovery and tools/build/openq4_version.py for the
# generated version header - so the file lists cannot drift apart. What does have
# to be kept in step by hand is compiler and link flags; the comments below mark
# the ones that are load-bearing rather than cosmetic.
#
#   cmake -B build -S . -DCMAKE_BUILD_TYPE=Release
#   cmake --build build

cmake_minimum_required(VERSION 3.20)
project(openQ4GameLibs VERSION 0.0.1 LANGUAGES CXX)

option(BUILD_SPGAME "Build the single-player game module" ON)
option(BUILD_MPGAME "Build the multiplayer game module" ON)
option(INLINE_DEBUG "Enable legacy DebugInline behavior for debug builds" OFF)
set(MSSDK_INCLUDE "" CACHE STRING
    "Optional legacy SDK include path relative to src/ (for example mssdk/include)")

find_package(Python3 COMPONENTS Interpreter REQUIRED)

# ---------------------------------------------------------------------------
# Host and toolchain gates, matching src/meson.build
# ---------------------------------------------------------------------------
if(CMAKE_SYSTEM_NAME STREQUAL "Windows")
  set(GAMELIBS_WINDOWS TRUE)
elseif(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
  set(GAMELIBS_DARWIN TRUE)
elseif(CMAKE_SYSTEM_NAME STREQUAL "Linux")
  set(GAMELIBS_LINUX TRUE)
else()
  message(FATAL_ERROR "openQ4-GameLibs supports Windows, macOS and Linux hosts.")
endif()

if(GAMELIBS_WINDOWS AND NOT MSVC)
  message(FATAL_ERROR "openQ4-GameLibs Windows builds require MSVC cl.exe.")
endif()
if(GAMELIBS_DARWIN AND NOT CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  message(FATAL_ERROR "openQ4-GameLibs macOS builds require Clang.")
endif()
if(GAMELIBS_LINUX AND NOT CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
  message(FATAL_ERROR "openQ4-GameLibs Linux builds require GCC or Clang.")
endif()

# Arch tag in the module filename. The engine resolves modules by this exact
# name, so it is not cosmetic.
set(_p "${CMAKE_SIZEOF_VOID_P}")
if(CMAKE_SYSTEM_PROCESSOR MATCHES "aarch64|arm64|ARM64")
  set(GAME_ARCH arm64)
elseif(_p EQUAL 8)
  set(GAME_ARCH x64)
elseif(_p EQUAL 4)
  set(GAME_ARCH x86)
else()
  message(FATAL_ERROR "Unsupported game module architecture.")
endif()

set(SP_MODULE_NAME "game-sp_${GAME_ARCH}")
set(MP_MODULE_NAME "game-mp_${GAME_ARCH}")

string(TOLOWER "${CMAKE_BUILD_TYPE}" _buildtype_lower)
if(_buildtype_lower MATCHES "^(release|relwithdebinfo|minsizerel)$")
  set(RELEASE_LIKE TRUE)
else()
  set(RELEASE_LIKE FALSE)
endif()

set(SOURCE_ROOT "${CMAKE_CURRENT_SOURCE_DIR}/src")

# ---------------------------------------------------------------------------
# Generated version header
#
# framework/licensee.h is kept byte-identical with the engine repo and includes
# this header, so it has to land in the build-dir counterpart of src/ for the
# existing include path to resolve it.
# ---------------------------------------------------------------------------
set(GENERATED_INCLUDE_DIR "${CMAKE_CURRENT_BINARY_DIR}/src")
file(MAKE_DIRECTORY "${GENERATED_INCLUDE_DIR}")

execute_process(
  COMMAND "${Python3_EXECUTABLE}"
          "${CMAKE_CURRENT_SOURCE_DIR}/tools/build/openq4_version.py"
          --source-root "${CMAKE_CURRENT_SOURCE_DIR}"
          --base-version "${PROJECT_VERSION}"
          --header-out "${GENERATED_INCLUDE_DIR}/openq4_version_generated.h"
  RESULT_VARIABLE _version_result
  OUTPUT_QUIET
)
if(NOT _version_result EQUAL 0)
  message(FATAL_ERROR "openq4_version.py failed (${_version_result})")
endif()

# ---------------------------------------------------------------------------
# Source discovery, through the repository's own lister
# ---------------------------------------------------------------------------
function(gamelibs_list_sources out_var subdir)
  execute_process(
    COMMAND "${Python3_EXECUTABLE}"
            "${SOURCE_ROOT}/buildscripts/list_sources.py"
            "${SOURCE_ROOT}" "${subdir}" ${ARGN}
    OUTPUT_VARIABLE _raw
    RESULT_VARIABLE _result
    OUTPUT_STRIP_TRAILING_WHITESPACE
  )
  if(NOT _result EQUAL 0)
    message(FATAL_ERROR "list_sources.py failed for '${subdir}' (${_result})")
  endif()
  string(REPLACE "\n" ";" _lines "${_raw}")
  set(_files)
  foreach(rel IN LISTS _lines)
    string(STRIP "${rel}" rel)
    if(rel)
      list(APPEND _files "${SOURCE_ROOT}/${rel}")
    endif()
  endforeach()
  set(${out_var} "${_files}" PARENT_SCOPE)
endfunction()

# The SDK's MSVC x86 inline-assembly SIMD sources. Simd.cpp selects the generic
# implementation on every other target supported here, so non-x86 compilers must
# not be asked to parse them.
set(IDLIB_EXCLUDES "idlib/math/Simd_AltiVec.cpp")
if(NOT GAME_ARCH STREQUAL "x86")
  list(APPEND IDLIB_EXCLUDES
    "idlib/math/Simd_3DNow.cpp"
    "idlib/math/Simd_MMX.cpp"
    "idlib/math/Simd_SSE.cpp"
    "idlib/math/Simd_SSE2.cpp"
    "idlib/math/Simd_SSE3.cpp")
endif()

gamelibs_list_sources(IDLIB_SOURCES idlib ${IDLIB_EXCLUDES})
gamelibs_list_sources(GAME_SOURCES game "game/Callbacks.cpp" "game/gamesys/Callbacks.cpp")
gamelibs_list_sources(MPGAME_SOURCES mpgame "mpgame/Callbacks.cpp" "mpgame/gamesys/Callbacks.cpp")

# ---------------------------------------------------------------------------
# Compile options
# ---------------------------------------------------------------------------
set(COMMON_CXX_FLAGS)
set(COMMON_DEFINES Q4SDK USE_OPENAL)

if(MSVC)
  list(APPEND COMMON_CXX_FLAGS
    /std:c++17 /W4 /we4302 /we4311 /we4312 /FS
    /source-charset:windows-1252 /execution-charset:windows-1252
    /Zc:strictStrings- /Zc:ternary-
    /wd4091 /wd4456 /wd4457 /wd4458 /wd4459 /wd4595 /wd4996 /wd4702 /wd4740 /wd4750)
  if(GAME_ARCH STREQUAL "x64")
    list(APPEND COMMON_CXX_FLAGS /we4101 /we4189 /we4267 /we4324 /we4505)
  endif()
  list(APPEND COMMON_DEFINES _WINDOWS WIN32)
else()
  include(CheckCXXCompilerFlag)
  list(APPEND COMMON_CXX_FLAGS -std=c++17)
  foreach(flag -fpermissive -Wno-invalid-source-encoding -Wno-deprecated-declarations
               -Wno-writable-strings -ffp-contract=off)
    # -ffp-contract=off is load-bearing, not a warning tweak: GCC and Clang
    # default to fast and fuse a*b+c into an FMA wherever the target has one.
    # AArch64 always does and the validated x86-64 SSE2 baseline does not, so
    # leaving it on makes identical game-side physics and AI math diverge per
    # architecture.
    string(MAKE_C_IDENTIFIER "HAVE${flag}" _flag_var)
    check_cxx_compiler_flag("${flag}" ${_flag_var})
    if(${_flag_var})
      list(APPEND COMMON_CXX_FLAGS "${flag}")
    endif()
  endforeach()
endif()

if(GAMELIBS_DARWIN)
  list(APPEND COMMON_DEFINES MACOS_X=1)
elseif(GAMELIBS_LINUX)
  list(APPEND COMMON_DEFINES ID_GL_HARDLINK)
  include(CheckCXXCompilerFlag)
  foreach(flag -fstack-protector-strong -fno-strict-aliasing)
    string(MAKE_C_IDENTIFIER "HAVE${flag}" _flag_var)
    check_cxx_compiler_flag("${flag}" ${_flag_var})
    if(${_flag_var})
      list(APPEND COMMON_CXX_FLAGS "${flag}")
    endif()
  endforeach()
  list(APPEND COMMON_DEFINES _FORTIFY_SOURCE=2)
  # Game modules consume engine-owned polymorphic interfaces whose key functions
  # and RTTI live in the executable, so UBSan's vptr metadata cannot satisfy the
  # modules' -z defs link contract across the DSO boundary. Every other
  # sanitizer check stays available.
  check_cxx_compiler_flag("-fno-sanitize=vptr" HAVE_NO_SANITIZE_VPTR)
  if(HAVE_NO_SANITIZE_VPTR)
    list(APPEND COMMON_CXX_FLAGS -fno-sanitize=vptr)
  endif()
endif()

# Deliberately no _DEBUG or NDEBUG outside release: the engine defines neither,
# and src/idlib/precompiled.h falls through to its own NDEBUG. Defining _DEBUG
# here changed shared struct layouts across the module boundary - srfTriangles_t
# grows description[64], Heap.h grows the MemScopedTag stack - so a debug module
# and a debug engine disagreed about the size of types they pass each other.
if(RELEASE_LIKE)
  list(APPEND COMMON_DEFINES NDEBUG)
endif()

if(INLINE_DEBUG AND NOT RELEASE_LIKE)
  list(APPEND COMMON_DEFINES _INLINEDEBUG)
  if(MSVC)
    list(APPEND COMMON_CXX_FLAGS /Ob1)
  endif()
endif()

if(MSSDK_INCLUDE)
  list(APPEND COMMON_CXX_FLAGS "-I${SOURCE_ROOT}/${MSSDK_INCLUDE}")
endif()

# Every game translation unit is compiled with idlib/precompiled.h force-included.
if(MSVC)
  set(FORCED_INCLUDE_FLAGS "/FI${SOURCE_ROOT}/idlib/precompiled.h")
else()
  set(FORCED_INCLUDE_FLAGS -include "${SOURCE_ROOT}/idlib/precompiled.h")
endif()

# ---------------------------------------------------------------------------
# idlib: one archive per game flavour
#
# idlib/precompiled.h includes mpgame/Game_local.h under GAME_MPAPI and
# game/Game_local.h otherwise, and every idlib translation unit includes
# precompiled.h. A single single-player-flavoured archive linked into both
# modules therefore put two different definitions of the same game types into
# each multiplayer binary.
# ---------------------------------------------------------------------------
if(GAMELIBS_WINDOWS)
  set(GAMELIBS_PIC OFF)
  set(GAMELIBS_IDLIB_VISIBILITY default)
else()
  set(GAMELIBS_PIC ON)
  set(GAMELIBS_IDLIB_VISIBILITY hidden)
endif()

function(gamelibs_add_idlib target_name)
  add_library(${target_name} STATIC ${IDLIB_SOURCES})
  target_include_directories(${target_name} PUBLIC "${SOURCE_ROOT}" "${GENERATED_INCLUDE_DIR}")
  target_compile_options(${target_name} PRIVATE ${COMMON_CXX_FLAGS})
  target_compile_definitions(${target_name} PRIVATE ${COMMON_DEFINES} GAME_DLL ${ARGN})
  set_target_properties(${target_name} PROPERTIES
    POSITION_INDEPENDENT_CODE ${GAMELIBS_PIC}
    CXX_VISIBILITY_PRESET ${GAMELIBS_IDLIB_VISIBILITY})
endfunction()

gamelibs_add_idlib(idLib)
gamelibs_add_idlib(idLibMP GAME_MPAPI)

# ---------------------------------------------------------------------------
# Module link options
# ---------------------------------------------------------------------------
set(MODULE_LINK_OPTIONS)
if(GAMELIBS_WINDOWS)
  list(APPEND MODULE_LINK_OPTIONS /FIXED:NO /LARGEADDRESSAWARE)
  # In lockstep with the engine repo: every 64-bit Windows target gets the
  # 64-bit preferred load address, only 32-bit x86 falls back.
  if(GAME_ARCH STREQUAL "x64" OR GAME_ARCH STREQUAL "arm64")
    list(APPEND MODULE_LINK_OPTIONS /BASE:0x180000000)
  else()
    list(APPEND MODULE_LINK_OPTIONS /BASE:0x20000000)
  endif()
  if(RELEASE_LIKE)
    list(APPEND MODULE_LINK_OPTIONS /STACK:4194304 /OPT:REF /OPT:ICF /MAP)
  else()
    list(APPEND MODULE_LINK_OPTIONS /STACK:16000000,16000000)
  endif()
elseif(GAMELIBS_DARWIN)
  # Darwin peer of the Linux version script and the Windows .def files: only the
  # module entry point may be exported.
  set(DARWIN_EXPORT_LIST "${CMAKE_CURRENT_SOURCE_DIR}/tools/build/darwin_game_module.exp")
  list(APPEND MODULE_LINK_OPTIONS
    -Wl,-dead_strip -Wl,-headerpad_max_install_names
    "-Wl,-exported_symbols_list,${DARWIN_EXPORT_LIST}")
elseif(GAMELIBS_LINUX)
  set(LINUX_EXPORT_MAP "${CMAKE_CURRENT_SOURCE_DIR}/tools/build/linux_game_module.map")
  list(APPEND MODULE_LINK_OPTIONS
    -Wl,-z,relro -Wl,-z,now -Wl,-z,noexecstack
    # -z defs is the contract that catches a module referencing an engine symbol
    # that no longer exists, at link time instead of at load time.
    -Wl,-z,defs
    "-Wl,--version-script=${LINUX_EXPORT_MAP}")
endif()

function(gamelibs_add_module target_name module_name idlib_target sources)
  add_library(${target_name} MODULE ${sources})
  target_link_libraries(${target_name} PRIVATE ${idlib_target})
  target_include_directories(${target_name} PRIVATE "${SOURCE_ROOT}" "${GENERATED_INCLUDE_DIR}")
  target_compile_options(${target_name} PRIVATE ${COMMON_CXX_FLAGS} ${FORCED_INCLUDE_FLAGS})
  target_compile_definitions(${target_name} PRIVATE ${COMMON_DEFINES} GAME_DLL ${ARGN})
  target_link_options(${target_name} PRIVATE ${MODULE_LINK_OPTIONS})
  set_target_properties(${target_name} PROPERTIES
    OUTPUT_NAME "${module_name}"
    PREFIX ""
    CXX_VISIBILITY_PRESET hidden
    POSITION_INDEPENDENT_CODE ${GAMELIBS_PIC})

  if(GAMELIBS_DARWIN)
    # A MODULE library on Darwin is an MH_BUNDLE with -undefined dynamic_lookup,
    # which carries no LC_ID_DYLIB for -install_name and drops the
    # undefined-symbol closure the other hosts get. Build a real dylib instead.
    set_target_properties(${target_name} PROPERTIES SUFFIX ".dylib")
    target_link_options(${target_name} PRIVATE
      "-Wl,-install_name,@loader_path/${module_name}.dylib")
    set_target_properties(${target_name} PROPERTIES LINK_DEPENDS "${DARWIN_EXPORT_LIST}")
  elseif(GAMELIBS_LINUX)
    set_target_properties(${target_name} PROPERTIES LINK_DEPENDS "${LINUX_EXPORT_MAP}")
  endif()
endfunction()

if(BUILD_SPGAME)
  gamelibs_add_module(game_sp "${SP_MODULE_NAME}" idLib "${GAME_SOURCES}")
  if(MSVC)
    target_sources(game_sp PRIVATE "${SOURCE_ROOT}/game/game.def")
  endif()
endif()

if(BUILD_MPGAME)
  gamelibs_add_module(game_mp "${MP_MODULE_NAME}" idLibMP "${MPGAME_SOURCES}" GAME_MPAPI)
  if(MSVC)
    target_sources(game_mp PRIVATE "${SOURCE_ROOT}/mpgame/mpgame.def")
  endif()
endif()

message(STATUS "openQ4-GameLibs (CMake)")
message(STATUS "  Arch          : ${GAME_ARCH}")
message(STATUS "  Build type    : ${CMAKE_BUILD_TYPE}")
message(STATUS "  SP module     : ${BUILD_SPGAME}")
message(STATUS "  MP module     : ${BUILD_MPGAME}")
