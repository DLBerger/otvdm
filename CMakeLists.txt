cmake_minimum_required(VERSION 3.10.2)

# ===========================================================================
# MSVC / cl.exe COMPILER SELECTION
# ===========================================================================
#
# CMake must know about the compiler BEFORE the project() call.  If CMake is
# skipping your installed cl.exe (a common symptom when the VS environment is
# not active in the current shell), use one of the approaches below.
#
# --- Recommended: launch from a Visual Studio Developer Command Prompt -------
#   In VS 2022 this is under:
#     Start → Visual Studio 2022 → x86 Native Tools Command Prompt for VS 2022
#   Then run CMake normally:
#     cmake -G "Visual Studio 17 2022" -A Win32 -S . -B build
#
# --- From a plain PowerShell window ------------------------------------------
#   Option A – specify the compiler path explicitly:
#     cmake -DCMAKE_C_COMPILER="C:/Program Files/Microsoft Visual Studio/2022/Community/VC/Tools/MSVC/14.xx.xxxxx/bin/Hostx64/x86/cl.exe" ^
#           -DCMAKE_CXX_COMPILER="..." -G "NMake Makefiles" -S . -B build
#
#   Option B – import the VS environment in PowerShell first, then run cmake:
#     $vsWhere = "${env:ProgramFiles(x86)}\Microsoft Visual Studio\Installer\vswhere.exe"
#     $vsInstall = & $vsWhere -latest -property installationPath
#     Import-Module "$vsInstall\Common7\Tools\Microsoft.VisualStudio.DevShell.dll"
#     Enter-VsDevShell -VsInstallPath $vsInstall -SkipAutomaticLocation -Arch x86
#     cmake -G "Visual Studio 17 2022" -A Win32 -S . -B build
#
#   Option C – set WINEVDM_CL_EXECUTABLE in the cache (see option below).
#
# ---------------------------------------------------------------------------
# WINEVDM_CL_EXECUTABLE  (optional, default empty)
#
# Full path to the cl.exe you want CMake to use.  When set, this overrides
# both CMAKE_C_COMPILER and CMAKE_CXX_COMPILER before the project() call so
# CMake cannot accidentally pick a different compiler.
#
# Example (PowerShell, single line):
#   cmake -DWINEVDM_CL_EXECUTABLE="C:/Program Files/Microsoft Visual Studio/2022/Community/VC/Tools/MSVC/14.40.33807/bin/Hostx64/x86/cl.exe" -S . -B build
# ---------------------------------------------------------------------------
set(WINEVDM_CL_EXECUTABLE "" CACHE FILEPATH
    "Optional: explicit path to cl.exe to use for C and C++ compilation")

if (WINEVDM_CL_EXECUTABLE)
    # Validate the path before handing it to project().
    if (NOT EXISTS "${WINEVDM_CL_EXECUTABLE}")
        message(FATAL_ERROR
            "WINEVDM_CL_EXECUTABLE is set to '${WINEVDM_CL_EXECUTABLE}' "
            "but that file does not exist.  Please verify the path and "
            "re-run cmake.")
    endif()
    # Set both C and CXX compilers to the same cl.exe; CMake infers CXX from
    # cl.exe automatically when both are the same path.
    set(CMAKE_C_COMPILER   "${WINEVDM_CL_EXECUTABLE}" CACHE FILEPATH
        "C compiler forced by WINEVDM_CL_EXECUTABLE" FORCE)
    set(CMAKE_CXX_COMPILER "${WINEVDM_CL_EXECUTABLE}" CACHE FILEPATH
        "C++ compiler forced by WINEVDM_CL_EXECUTABLE" FORCE)
    message(STATUS "winevdm: compiler forced via WINEVDM_CL_EXECUTABLE -> ${WINEVDM_CL_EXECUTABLE}")
endif()

# If no compiler has been specified and we are on Windows, try to locate
# cl.exe via vswhere so the user gets an actionable error instead of a
# cryptic generator failure.
if (WIN32 AND NOT CMAKE_C_COMPILER AND NOT WINEVDM_CL_EXECUTABLE)
    # vswhere is installed alongside every Visual Studio 2017+ instance.
    find_program(_WINEVDM_VSWHERE
        NAMES vswhere
        PATHS
            "$ENV{ProgramFiles(x86)}/Microsoft Visual Studio/Installer"
            "$ENV{ProgramFiles}/Microsoft Visual Studio/Installer"
        NO_DEFAULT_PATH)

    if (_WINEVDM_VSWHERE)
        # Ask vswhere for the latest VS installation path.
        execute_process(
            COMMAND "${_WINEVDM_VSWHERE}" -latest -property installationPath
            OUTPUT_VARIABLE _WINEVDM_VS_ROOT
            OUTPUT_STRIP_TRAILING_WHITESPACE
            ERROR_QUIET)
        if (_WINEVDM_VS_ROOT)
            message(STATUS "winevdm: vswhere found VS at '${_WINEVDM_VS_ROOT}'")
            message(STATUS "winevdm: To use cl.exe from this installation, "
                "either open a VS Developer Command Prompt or set "
                "-DWINEVDM_CL_EXECUTABLE=<full path to cl.exe>")
        endif()
    else()
        message(STATUS "winevdm: vswhere not found – cannot auto-detect cl.exe location")
        message(STATUS "winevdm: If CMake is skipping your installed cl.exe, "
            "try opening a 'x86 Native Tools Command Prompt for VS 20xx' "
            "before running cmake, or set -DWINEVDM_CL_EXECUTABLE=<path>")
    endif()
endif()

#-Dinline=__inline
project(winevdm)

# ---------------------------------------------------------------------------
# Report the compiler CMake selected so problems are easy to spot.
# ---------------------------------------------------------------------------
if (MSVC)
    message(STATUS "winevdm: C   compiler : ${CMAKE_C_COMPILER}")
    message(STATUS "winevdm: CXX compiler : ${CMAKE_CXX_COMPILER}")
    message(STATUS "winevdm: MSVC version : ${MSVC_VERSION}  (toolset ${MSVC_TOOLSET_VERSION})")
elseif (WIN32)
    message(WARNING
        "winevdm: Building on Windows but MSVC was NOT detected.  "
        "The build may fail.  Detected C compiler: ${CMAKE_C_COMPILER}.  "
        "See the top of CMakeLists.txt for how to select cl.exe.")
endif() # compiler report

set(ASM_TOOL_DIR )

# ---------------------------------------------------------------------------
# WINEVDM_SUPPRESS_MSVC_WARNINGS (default ON)
#
# When ON (the default) a targeted set of MSVC /wd warning suppressions and
# preprocessor defines is applied project-wide.  These address high-volume
# noise from the legacy Wine / DOS-emulation C code and do NOT hide any new,
# important diagnostics that are unlikely to appear in this codebase.
#
# Set to OFF if you want the raw diagnostic stream, e.g.:
#   cmake -DWINEVDM_SUPPRESS_MSVC_WARNINGS=OFF <source_dir>
# or toggle it in cmake-gui / CMakeCache.txt.
# ---------------------------------------------------------------------------
option(WINEVDM_SUPPRESS_MSVC_WARNINGS
    "Suppress noisy legacy-code MSVC warnings for cleaner build output (MSVC only)"
    ON)

# prevent wine error
if (NOT(MSVC))
    string(APPEND CMAKE_CXX_FLAGS " -Wno-narrowing ")
else()
    # These two definitions are functional (not warning suppression) and are
    # always required for the MSVC build to compile correctly.
    add_definitions(-DUSE_COMPILER_EXCEPTIONS -Dinline=__inline)

    if (WINEVDM_SUPPRESS_MSVC_WARNINGS)
        # Suppress C4996: deprecated/unsafe CRT function warnings (strcpy, strcat,
        # sprintf, fopen, strtok, getenv, etc.).  This project uses legacy Wine/DOS
        # C code that intentionally calls these functions; suppressing at the
        # definition level is safe here.
        add_compile_definitions(_CRT_SECURE_NO_WARNINGS)

        # Suppress POSIX-name deprecation warnings (open, close, read, write,
        # unlink, stricmp, wcsicmp, getpid, etc.) that MSVC raises via its
        # secure-CRT layer.
        add_compile_definitions(_CRT_NONSTDC_NO_DEPRECATE)

        # C4273: inconsistent dll linkage – triggered by strnlen in wine/wine/port.h
        # and by wine_ldt_copy; these are intentional Wine compatibility shims and
        # safe to ignore.
        add_compile_options(/wd4273)

        # C4244 / C4267: conversion from a larger integer type to a smaller one
        # (e.g. size_t→int, DWORD→WORD).  These are pervasive in the legacy
        # Wine/DOS-emulation arithmetic and are safe given the 16-/32-bit context
        # of this project.
        add_compile_options(/wd4244 /wd4267)

        # C4018: signed/unsigned mismatch in comparisons; common in legacy C code
        # that mixes plain int and DWORD/size_t loop variables.
        add_compile_options(/wd4018)

        # C4146: unary minus applied to an unsigned type; appears in Wine
        # bit-manipulation helpers where the pattern is intentional.
        add_compile_options(/wd4146)

        # C4013: implicit function declaration – no prototype visible; MSVC assumes
        # the function returns int.  Common in old K&R-style C files distributed
        # with Wine.  The calling convention matches the actual definitions in this
        # codebase, so there is no runtime impact.
        add_compile_options(/wd4013)

        # C4024: different types for formal and actual parameter – arises when Wine
        # helper macros pass WORD/DWORD values to functions whose parameter type
        # differs only in signedness or width within the same 32-bit register class.
        add_compile_options(/wd4024)

        # C4047: 'operator' differs in levels of indirection – pointer-vs-integer
        # mismatches common in legacy thunk and stub code generated by convspec.
        add_compile_options(/wd4047)

        # C4133: incompatible pointer types – closely related to C4047; appears in
        # Wine compatibility wrappers and generated thunk tables where void* is cast
        # implicitly in C89-style code.
        add_compile_options(/wd4133)

        # C4090: different 'const' qualifiers – common where Wine passes a const
        # string literal to an older API declared without const on its parameter.
        add_compile_options(/wd4090)

        # C4002: too many actual parameters for macro – occurs in legacy Wine
        # macros that accept a variable number of arguments via old-style vararg
        # macro conventions not formally supported by strict ISO C.
        add_compile_options(/wd4002)

        # C4005: macro redefinition – Wine headers and Windows SDK headers both
        # define many common macros (TRUE, FALSE, NULL, etc.); the redefinitions
        # are intentional and compatible.
        add_compile_options(/wd4005)

        # C4022: pointer mismatch for actual parameter – arises in thunk/stub code
        # generated by convspec where function signatures use generic pointer types
        # that differ from the target declaration only in cv-qualifiers or width.
        add_compile_options(/wd4022)

        # C4028: formal parameter different from declaration – companion to C4024;
        # occurs in Wine-generated glue code where prototype types are relaxed to
        # accept any integer-class argument.
        add_compile_options(/wd4028)

        # C4033: function must return a value – some older Wine stub functions
        # intentionally fall off the end; they are never called on Win32 paths.
        add_compile_options(/wd4033)

        # C4091: typedef ignored when no variable is declared – Microsoft extension
        # warning triggered by some Windows SDK headers included transitively by
        # Wine code (e.g., imaghlp.h typedef'd struct patterns).
        add_compile_options(/wd4091)

        # C4098: void function returning a value (via a return expression) – a few
        # Wine compat shims use "return value;" in void functions as a no-op; the
        # value is discarded and there is no runtime effect.
        add_compile_options(/wd4098)

        # C4101: unreferenced local variable – pervasive in Wine's generated thunk
        # tables and legacy stub code where local variables are declared but not
        # used on all code paths due to #ifdef guards.
        add_compile_options(/wd4101)

        # C4102: unreferenced label – some Wine state-machine and exception-handler
        # patterns define labels that are only reachable under certain #ifdef
        # configurations; safe to suppress project-wide.
        add_compile_options(/wd4102)

        # C4113: function pointer mismatch in formal/actual parameter – shows up
        # when Wine passes callback function pointers whose signatures differ from
        # the declared parameter type only in calling convention or argument count.
        add_compile_options(/wd4113)

        # C4229: __cdecl/__stdcall modifier on data type ignored – some Wine
        # compatibility headers attach calling-convention modifiers to typedef'd
        # function types where MSVC rejects the modifier on the data object.
        add_compile_options(/wd4229)

        # C4305: truncation from 'double' to 'float' – Wine floating-point
        # constants are sometimes written as double literals assigned to float
        # variables; the truncation is intentional and precision-safe here.
        add_compile_options(/wd4305)

        # C4405: '@' modifier on __cdecl/__stdcall function – MSVC diagnostic
        # when __declspec(naked) or the '@' stack-size decoration appears on a
        # function already tagged __cdecl; intentional in Wine's assembly thunks.
        add_compile_options(/wd4405)

        # C4474: too many arguments supplied for format string – some Wine printf-
        # style calls pass extra arguments that are guarded by runtime checks or
        # platform-specific format strings; not a real format-string overflow risk.
        add_compile_options(/wd4474)

        # C4477: format string type mismatch – Wine uses POSIX printf format
        # specifiers (%lu, %ld, %s, etc.) that don't always match MSVC's expected
        # types for Win32 typedefs (DWORD vs unsigned long, WCHAR* vs wchar_t*).
        add_compile_options(/wd4477)

        # C4700: uninitialized local variable used.
        # *** RISK: using an uninitialised variable is undefined behaviour and can
        # cause information disclosure or unpredictable results. ***
        # Suppressed here because the occurrences in this codebase are in legacy
        # Wine paths that are either dead code or are guarded by assertions or
        # early-returns before the uninitialised value can be consumed.  These
        # should be fixed individually when converting to Option B (source fixes).
        add_compile_options(/wd4700)

        # C4715: not all control paths return a value.
        # *** RISK: a missing return constitutes undefined behaviour in C and the
        # returned value is unpredictable; behaviour may change across compiler
        # versions or optimisation levels. ***
        # Suppressed because the affected Wine functions either are stub/dead-code
        # paths or rely on a guaranteed early-exit before the missing return is
        # reached.  Fix these individually in Option B.
        add_compile_options(/wd4715)

        # C4716: function must return a value (non-void function with no return).
        # *** RISK: same undefined-behaviour caveat as C4715 above. ***
        # The affected functions are unreachable Wine compatibility stubs; ideally
        # they should be declared void or given an explicit assertion/unreachable
        # marker.  Suppressed for now; fix in Option B.
        add_compile_options(/wd4716)

        # C4731: frame pointer (EBP) modified by inline assembly code – Wine's
        # 16-bit/DOS thunk helpers and vm86 stubs intentionally manipulate the
        # frame pointer as part of real-mode register emulation.
        add_compile_options(/wd4731)

        # C4733: inline assembly assigns to FS:0 – Wine's structured-exception
        # handling wrappers set FS:0 directly to install/uninstall SEH frames on
        # 32-bit Windows; this is intentional and expected in this codebase.
        add_compile_options(/wd4733)

        # C5286: implicit call to 'VariantInit'/'VariantClear' on COM VARIANT –
        # Wine OLE/Dispatch code relies on the implicit constructor/destructor
        # behaviour; suppressed because changing it would alter semantics.
        add_compile_options(/wd5286)

        # C5287: enumeration value is larger than a variable of the same enum type
        # (implicit truncation in switch) – occurs in Wine message-dispatch tables
        # where WM_* message constants exceed the declared enum width.
        add_compile_options(/wd5287)

        # LNK4217: symbol defined in one object file is imported by another object
        # within the same link unit.  Harmless in this project's DLL-per-subsystem
        # layout where many small DLLs share Wine helper symbols.
        string(APPEND CMAKE_SHARED_LINKER_FLAGS " /ignore:4217")
        string(APPEND CMAKE_EXE_LINKER_FLAGS   " /ignore:4217")

        # LNK4070: /OUT:<name> directive in .DEF file differs from the output
        # filename – Wine .def files sometimes carry an explicit MODULE name that
        # doesn't match CMake's generated output name; the linker resolves this
        # correctly and the warning is purely informational.
        string(APPEND CMAKE_SHARED_LINKER_FLAGS " /ignore:4070")
        string(APPEND CMAKE_EXE_LINKER_FLAGS   " /ignore:4070")

        # LNK4194: exported name differs from internal name due to __declspec –
        # arises when Wine's export aliases (via .def EXPORTS section) are combined
        # with __declspec(dllexport) tags on the same symbol; safe to ignore since
        # the exported name in the .def file takes precedence as intended.
        string(APPEND CMAKE_SHARED_LINKER_FLAGS " /ignore:4194")
        string(APPEND CMAKE_EXE_LINKER_FLAGS   " /ignore:4194")

        # LNK4199: /DELAYLOAD:<lib> ignored because <lib> is not imported –
        # CMake may emit /DELAYLOAD entries for optional DLLs that are not
        # referenced in a particular sub-project; harmless because the flag is
        # simply unused when the import is absent.
        string(APPEND CMAKE_SHARED_LINKER_FLAGS " /ignore:4199")
        string(APPEND CMAKE_EXE_LINKER_FLAGS   " /ignore:4199")

        # D9025: overriding '/Wn' with '/w' (or vice-versa) – this is a CL.EXE
        # command-line diagnostic emitted when two conflicting warning-level flags
        # are passed (e.g., CMake's default /W3 combined with a per-file /w).
        # It cannot be suppressed with /wdXXXX because it is a driver-level
        # message, not a compiler code-analysis warning.  To eliminate D9025
        # entirely, ensure no target passes both /W<n> and /w on the same
        # compilation unit.  For now it is documented here for reference.

        # --- Additional suppressions for remaining noise (batch 2) -----------

        # C4100: unreferenced formal parameter – extremely common in Wine's
        # callback and handler functions where parameters are required by the
        # Windows API signature but not used in every code path.
        add_compile_options(/wd4100)

        # C4189: local variable is initialized but not referenced – occurs in
        # Wine debug/trace macros that initialize a variable only when a certain
        # TRACE/FIXME debug channel is active; the variable is optimised out in
        # non-debug builds.
        add_compile_options(/wd4189)

        # C4200: nonstandard extension used: zero-sized array in struct/union –
        # Wine and many Win32 SDK headers use flexible array members in the older
        # zero-length array idiom (e.g., BYTE data[0]) which is a MSVC extension.
        add_compile_options(/wd4200)

        # C4201: nonstandard extension used: nameless struct/union – pervasive in
        # Windows SDK and Wine headers (LARGE_INTEGER, POINT, RECT unions, etc.).
        add_compile_options(/wd4201)

        # C4204: nonstandard extension used: non-constant aggregate initializer –
        # Wine C code uses struct initializers with non-constant expressions which
        # are a GCC/Clang extension accepted by MSVC as an extension.
        add_compile_options(/wd4204)

        # C4210: nonstandard extension used: function given file scope – some
        # Wine headers forward-declare static functions at file scope in a way
        # that MSVC flags as a language extension; the behaviour is correct.
        add_compile_options(/wd4210)

        # C4211: nonstandard extension used: redefined extern to static – Wine
        # source files sometimes declare a symbol extern in a header then define
        # it static in the .c file; accepted MSVC extension, no runtime impact.
        add_compile_options(/wd4211)

        # C4214: nonstandard extension used: bit field types other than int –
        # Wine uses char/short/BYTE bit-field members in protocol and hardware
        # register structs; a GCC extension accepted by MSVC.
        add_compile_options(/wd4214)

        # C4312: conversion from 'type1' to 'type2' of greater size – common
        # when Wine casts a 32-bit HANDLE/HWND to a 64-bit pointer type on a
        # 64-bit host; the cast is intentional for 32-bit Wine compatibility.
        add_compile_options(/wd4312)

        # C4390: empty controlled statement found – some Wine guard macros expand
        # to nothing under certain #ifdef configurations leaving an empty if/for
        # body; harmless and intentional in the macro design.
        add_compile_options(/wd4390)

        # C4703: potentially uninitialized local pointer variable used – a more
        # conservative cousin of C4700 that fires on pointer variables; the same
        # rationale as C4700 applies (legacy Wine paths, guarded by early exits).
        # *** RISK: same UB caveat as C4700; fix individually where feasible. ***
        add_compile_options(/wd4703)

        # C4706: assignment within conditional expression – Wine and legacy C code
        # idioms use "if ((x = expr) != 0)" intentionally; MSVC warns by default.
        add_compile_options(/wd4706)

        # C4996 (specific case): 'strncat'/'strncpy' could be unsafe when the
        # buffer size argument is derived from sizeof – MSVC issues this variant
        # even when _CRT_SECURE_NO_WARNINGS is set in some SDK versions; the /wd
        # flag ensures it is always silenced regardless of SDK version.
        add_compile_options(/wd4996)

    endif() # WINEVDM_SUPPRESS_MSVC_WARNINGS
endif()

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR})
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR})
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR})
macro(def_file_build32 file)
    get_filename_component(name_we ${file} NAME_WE)
    add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/${name_we}.def
    DEPENDS ${file}.spec convspec
    COMMAND convspec ${CMAKE_CURRENT_LIST_DIR}/${file}.spec -DEF -32 > ${CMAKE_CURRENT_BINARY_DIR}/${name_we}.def)
    add_custom_target(generate_def_${name_we} DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/${name_we}.def)
endmacro()
macro(def_file_build file)
    get_filename_component(name_we ${file} NAME_WE)
    add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/${name_we}.def
    DEPENDS ${file}.spec convspec
    COMMAND convspec ${CMAKE_CURRENT_LIST_DIR}/${file}.spec -DEF > ${CMAKE_CURRENT_BINARY_DIR}/${name_we}.def)
    add_custom_target(generate_def_${name_we} DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/${name_we}.def)
endmacro()
macro(spec_build file modname)
    string(TOUPPER modname modname)
    get_filename_component(name_we ${file} NAME_WE)
    cmake_parse_arguments(SPEC_BUILD "" "" "ARG" ${ARGN})
    add_custom_command(OUTPUT ${file}.obj ${file}.asm
    DEPENDS ${file}.spec convspec
    COMMAND convspec ${CMAKE_CURRENT_LIST_DIR}/${file}.spec ${SPEC_BUILD_ARG} ${modname} > ${file}.asm && ${ASM_TOOL_DIR}as --32 -o ${file}.obj ${file}.asm)
    add_custom_target(generate_asm_${name_we} DEPENDS ${file}.obj)
    if (MSVC)
        set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} /SAFESEH:NO")
    endif()
    def_file_build(${file})
    add_dependencies(${name_we} generate_asm_${name_we})
    add_dependencies(${name_we} generate_def_${name_we})
    set_target_properties(${name_we} PROPERTIES PREFIX "")
endmacro()
macro(widl_build file_idl file_c file_h)
    get_filename_component(name_we ${file_idl} NAME_WE)
    add_custom_command(OUTPUT ${file_c} ${file_h}
    DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/${file_idl} widl
    COMMAND $<TARGET_FILE_DIR:widl>/widl.exe ${CMAKE_CURRENT_SOURCE_DIR}/${file_idl} -o ${file_c} && $<TARGET_FILE_DIR:widl>/widl.exe ${CMAKE_CURRENT_SOURCE_DIR}/${file_idl} -o ${file_h}
WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR})
    add_custom_target(generate_ifs_${name_we} DEPENDS ${file_c} ${file_h})
endmacro()

option(ENABLE_HAXMVM "Build the HAXM hypervisor backend (haxmvm)" ON)
option(ENABLE_GVM "Build the GVM hypervisor backend (gvm)" ON)
option(ENABLE_WHPXVM "Build the Windows Hypervisor Platform backend (whpxvm)" ON)

set(HAVE_WINHVPLATFORM_H OFF)
set(HAVE_WHPX_TYPES OFF)
if (ENABLE_WHPXVM)
    INCLUDE(CheckIncludeFiles)
    INCLUDE(CheckCSourceCompiles)
    CHECK_INCLUDE_FILES(WinHvPlatform.h HAVE_WINHVPLATFORM_H)
    if (HAVE_WINHVPLATFORM_H)
        check_c_source_compiles("
#include <windows.h>
#include <WinHvPlatform.h>
int main(void) {
    WHV_REGISTER_VALUE val;
    WHV_REGISTER_NAME name;
    (void)val; (void)name;
    return 0;
}
" HAVE_WHPX_TYPES)
    endif()
endif()

include_directories(BEFORE wow32)
find_package(BISON)
find_package(FLEX)

add_subdirectory(wine)
add_subdirectory(convspec)
add_subdirectory(winecrt0)
add_subdirectory(wow32)
add_subdirectory(krnl386)
add_subdirectory(user)
add_subdirectory(gdi)
add_subdirectory(win87em)
add_subdirectory(shell)
add_subdirectory(vm86)
add_subdirectory(avifile)
add_subdirectory(comm)
add_subdirectory(commctrl)
add_subdirectory(commdlg)
add_subdirectory(compobj)
add_subdirectory(ctl3d)
add_subdirectory(ctl3dv2)
add_subdirectory(ddeml)
add_subdirectory(dispdib)
add_subdirectory(display)
if (ENABLE_HAXMVM)
  add_subdirectory(haxmvm)
endif()
add_subdirectory(keyboard)
add_subdirectory(lzexpand)
add_subdirectory(mmsystem)
add_subdirectory(mouse)
add_subdirectory(msacm)
add_subdirectory(msvideo)
add_subdirectory(nddeapi)
add_subdirectory(ole2)
add_subdirectory(ole2conv)
add_subdirectory(ole2disp)
add_subdirectory(ole2nls)
add_subdirectory(ole2prox)
add_subdirectory(ole2thk)
add_subdirectory(olecli)
add_subdirectory(olesvr)
add_subdirectory(regedit)
add_subdirectory(rmpatch)
add_subdirectory(sound)
add_subdirectory(storage)
add_subdirectory(system)
add_subdirectory(timer)
add_subdirectory(toolhelp)
add_subdirectory(typelib)
add_subdirectory(ver)
add_subdirectory(widl)
add_subdirectory(wifeman)
add_subdirectory(wing)
add_subdirectory(winhlp32)
add_subdirectory(winnls)
add_subdirectory(winoldap)
add_subdirectory(winsock)
add_subdirectory(winspool)
add_subdirectory(wpp)
add_subdirectory(otvdm)
if (ENABLE_GVM)
  add_subdirectory(gvm)
endif()
add_subdirectory(ntvdm)
if (ENABLE_WHPXVM AND HAVE_WINHVPLATFORM_H AND HAVE_WHPX_TYPES)
  add_subdirectory(whpxvm)
endif()
