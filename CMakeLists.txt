# CMake build file list for OpenAL

cmake_minimum_required(VERSION 3.0.2)

project(OpenAL)

if (COMMAND cmake_policy)
    cmake_policy(SET CMP0003 NEW)
    cmake_policy(SET CMP0005 NEW)
    if (POLICY CMP0020)
        cmake_policy(SET CMP0020 NEW)
    endif (POLICY CMP0020)
    if (POLICY CMP0042)
        cmake_policy(SET CMP0042 NEW)
    endif (POLICY CMP0042)
    if (POLICY CMP0054)
        cmake_policy(SET CMP0054 NEW)
    endif (POLICY CMP0054)
endif (COMMAND cmake_policy)

set(CMAKE_MODULE_PATH "${OpenAL_SOURCE_DIR}/cmake")

include(CheckFunctionExists)
include(CheckLibraryExists)
include(CheckSharedFunctionExists)
include(CheckIncludeFile)
include(CheckIncludeFiles)
include(CheckSymbolExists)
include(CheckCCompilerFlag)
include(CheckCXXCompilerFlag)
include(CheckCSourceCompiles)
include(CheckTypeSize)
include(CheckStructHasMember)
include(CheckFileOffsetBits)
include(GNUInstallDirs)

set(CMAKE_ALLOW_LOOSE_LOOP_CONSTRUCTS TRUE)


option(ALSOFT_DLOPEN  "Check for the dlopen API for loading optional libs"  ON)

option(ALSOFT_WERROR  "Treat compile warnings as errors"      OFF)

option(ALSOFT_UTILS          "Build and install utility programs"         ON)
option(ALSOFT_NO_CONFIG_UTIL "Disable building the alsoft-config utility" OFF)

option(ALSOFT_EXAMPLES  "Build and install example programs"  ON)
option(ALSOFT_TESTS     "Build and install test programs"     ON)

option(ALSOFT_CONFIG "Install alsoft.conf sample configuration file" ON)
option(ALSOFT_HRTF_DEFS "Install HRTF definition files" ON)
option(ALSOFT_AMBDEC_PRESETS "Install AmbDec preset files" ON)
option(ALSOFT_INSTALL "Install headers and libraries" ON)

if (DEFINED SHARE_INSTALL_DIR)
    message(WARNING "SHARE_INSTALL_DIR is deprecated.  Use the variables provided by the GNUInstallDirs module instead")
    set(CMAKE_INSTALL_DATADIR "${SHARE_INSTALL_DIR}")
endif ()

if (DEFINED LIB_SUFFIX)
    message(WARNING "LIB_SUFFIX is deprecated.  Use the variables provided by the GNUInstallDirs module instead")
endif ()


set(CPP_DEFS ) # C pre-process, not C++
set(INC_PATHS )
set(C_FLAGS )
set(LINKER_FLAGS )
set(EXTRA_LIBS )

if (WIN32)
    set(CPP_DEFS ${CPP_DEFS} _WIN32 _WIN32_WINNT=0x0502)

    option(ALSOFT_BUILD_ROUTER  "Build the router (EXPERIMENTAL; creates OpenAL32.dll and soft_oal.dll)"  OFF)

    # This option is mainly for static linking OpenAL Soft into another project
    # that already defines the IDs. It is up to that project to ensure all
    # required IDs are defined.
    option(ALSOFT_NO_UID_DEFS "Do not define GUIDs, IIDs, CLSIDs, or PropertyKeys" OFF)

    if (MINGW)
        option(ALSOFT_BUILD_IMPORT_LIB "Build an import .lib using dlltool (requires sed)" ON)
        if (NOT DLLTOOL)
            if (HOST)
                set(DLLTOOL "${HOST}-dlltool")
            else ()
                set(DLLTOOL "dlltool")
            endif ()
        endif ()
    endif ()
endif ()


# QNX's gcc do not uses /usr/include and /usr/lib pathes by default
if ("${CMAKE_C_PLATFORM_ID}" STREQUAL "QNX")
    set(INC_PATHS ${INC_PATHS} /usr/include)
    set(LINKER_FLAGS ${LINKER_FLAGS} -L/usr/lib)
endif ()

if (NOT LIBTYPE)
    set(LIBTYPE SHARED)
endif ()

set(LIB_MAJOR_VERSION "1")
set(LIB_MINOR_VERSION "19")
set(LIB_REVISION "1")
set(LIB_VERSION "${LIB_MAJOR_VERSION}.${LIB_MINOR_VERSION}.${LIB_REVISION}")

set(EXPORT_DECL "")
set(ALIGN_DECL "")


CHECK_TYPE_SIZE("long" SIZEOF_LONG)
CHECK_TYPE_SIZE("long long" SIZEOF_LONG_LONG)


check_c_compiler_flag(-std=c11 HAVE_STD_C11)
if (HAVE_STD_C11)
    set(CMAKE_C_FLAGS "-std=c11 ${CMAKE_C_FLAGS}")
else ()
    check_c_compiler_flag(-std=c99 HAVE_STD_C99)
    if (HAVE_STD_C99)
        set(CMAKE_C_FLAGS "-std=c99 ${CMAKE_C_FLAGS}")
    endif ()
endif ()

CHECK_CXX_COMPILER_FLAG(-std=c++11 HAVE_STD_CXX11)
if (HAVE_STD_CXX11)
    set(CMAKE_CXX_FLAGS "-std=c++11 ${CMAKE_CXX_FLAGS}")
endif ()

if (NOT WIN32)
    # Check if _POSIX_C_SOURCE and _XOPEN_SOURCE needs to be set for POSIX functions
    check_symbol_exists(posix_memalign stdlib.h HAVE_POSIX_MEMALIGN_DEFAULT)
    if (NOT HAVE_POSIX_MEMALIGN_DEFAULT)
        set(OLD_REQUIRED_FLAGS ${CMAKE_REQUIRED_FLAGS})
        set(CMAKE_REQUIRED_FLAGS "${CMAKE_REQUIRED_FLAGS} -D_POSIX_C_SOURCE=200112L -D_XOPEN_SOURCE=600")
        check_symbol_exists(posix_memalign stdlib.h HAVE_POSIX_MEMALIGN_POSIX)
        if (NOT HAVE_POSIX_MEMALIGN_POSIX)
            set(CMAKE_REQUIRED_FLAGS ${OLD_REQUIRED_FLAGS})
        else ()
            set(CPP_DEFS ${CPP_DEFS} _POSIX_C_SOURCE=200112L _XOPEN_SOURCE=600)
        endif ()
    endif ()
    unset(OLD_REQUIRED_FLAGS)
endif ()

# Set defines for large file support
CHECK_FILE_OFFSET_BITS()
if (_FILE_OFFSET_BITS)
    set(CPP_DEFS ${CPP_DEFS} "_FILE_OFFSET_BITS=${_FILE_OFFSET_BITS}")
    set(CMAKE_REQUIRED_FLAGS "${CMAKE_REQUIRED_FLAGS} -D_FILE_OFFSET_BITS=${_FILE_OFFSET_BITS}")
endif ()
set(CPP_DEFS ${CPP_DEFS} _LARGEFILE_SOURCE _LARGE_FILES)
set(CMAKE_REQUIRED_FLAGS "${CMAKE_REQUIRED_FLAGS} -D_LARGEFILE_SOURCE -D_LARGE_FILES")

# MSVC may need workarounds for C99 restrict and inline
if (MSVC)
    # TODO: Once we truly require C99, these restrict and inline checks should go
    # away.
    check_c_source_compiles("int *restrict foo;
                             int main() {return 0;}" HAVE_RESTRICT)
    if (NOT HAVE_RESTRICT)
        set(CPP_DEFS ${CPP_DEFS} "restrict=")
        set(CMAKE_REQUIRED_FLAGS "${CMAKE_REQUIRED_FLAGS} -Drestrict=")
    endif ()

    check_c_source_compiles("inline void foo(void) { }
                             int main() {return 0;}" HAVE_INLINE)
    if (NOT HAVE_INLINE)
        check_c_source_compiles("__inline void foo(void) { }
                                 int main() {return 0;}" HAVE___INLINE)
        if (NOT HAVE___INLINE)
            message(FATAL_ERROR "No inline keyword found, please report!")
        endif ()

        set(CPP_DEFS ${CPP_DEFS} inline=__inline)
        set(CMAKE_REQUIRED_FLAGS "${CMAKE_REQUIRED_FLAGS} -Dinline=__inline")
    endif ()
endif ()

# Make sure we have C99-style inline semantics with GCC (4.3 or newer).
if (CMAKE_COMPILER_IS_GNUCC)
    set(CMAKE_C_FLAGS "-fno-gnu89-inline ${CMAKE_C_FLAGS}")

    set(OLD_REQUIRED_FLAGS "${CMAKE_REQUIRED_FLAGS}")
    # Force no inlining for the next test.
    set(CMAKE_REQUIRED_FLAGS "${OLD_REQUIRED_FLAGS} -fno-inline")

    check_c_source_compiles("extern inline int foo() { return 0; }
                             int main() {return foo();}" INLINE_IS_C99)
    if (NOT INLINE_IS_C99)
        message(FATAL_ERROR "Your compiler does not seem to have C99 inline semantics!
                             Please update your compiler for better C99 compliance.")
    endif ()

    set(CMAKE_REQUIRED_FLAGS "${OLD_REQUIRED_FLAGS}")
endif ()

# Check if we have a proper timespec declaration
CHECK_STRUCT_HAS_MEMBER("struct timespec" tv_sec time.h HAVE_STRUCT_TIMESPEC)
if (HAVE_STRUCT_TIMESPEC)
    # Define it here so we don't have to include config.h for it
    set(CPP_DEFS ${CPP_DEFS} HAVE_STRUCT_TIMESPEC)
endif ()

# Some systems may need libatomic for C11 atomic functions to work
set(OLD_REQUIRED_LIBRARIES ${CMAKE_REQUIRED_LIBRARIES})
set(CMAKE_REQUIRED_LIBRARIES ${OLD_REQUIRED_LIBRARIES} atomic)
check_c_source_compiles("#include <stdatomic.h>
int _Atomic foo = ATOMIC_VAR_INIT(0);
int main()
{
    return atomic_fetch_add(&foo, 2);
}"
HAVE_LIBATOMIC)
if (NOT HAVE_LIBATOMIC)
    set(CMAKE_REQUIRED_LIBRARIES "${OLD_REQUIRED_LIBRARIES}")
else ()
    set(EXTRA_LIBS atomic ${EXTRA_LIBS})
endif ()
unset(OLD_REQUIRED_LIBRARIES)

# Include liblog for Android logging
check_library_exists(log __android_log_print "" HAVE_LIBLOG)
if (HAVE_LIBLOG)
    set(EXTRA_LIBS log ${EXTRA_LIBS})
    set(CMAKE_REQUIRED_LIBRARIES ${CMAKE_REQUIRED_LIBRARIES} log)
endif ()

# Check if we have C99 bool
check_c_source_compiles(
"int main(int argc, char *argv[])
 {
     volatile _Bool ret;
     ret = (argc > 1) ? 1 : 0;
     return ret ? -1 : 0;
 }"
HAVE_C99_BOOL)

# Check if we have C11 static_assert
check_c_source_compiles(
"int main()
 {
     _Static_assert(sizeof(int) == sizeof(int), \"What\");
     return 0;
 }"
HAVE_C11_STATIC_ASSERT)

# Check if we have C11 alignas
check_c_source_compiles(
"_Alignas(16) int foo;
 int main()
 {
     return 0;
 }"
HAVE_C11_ALIGNAS)

# Check if we have C11 _Atomic
check_c_source_compiles(
"#include <stdatomic.h>
 int _Atomic foo = ATOMIC_VAR_INIT(0);
 int main()
 {
     atomic_fetch_add(&foo, 2);
     return 0;
 }"
HAVE_C11_ATOMIC)

# Add definitions, compiler switches, etc.
include_directories("${OpenAL_SOURCE_DIR}/include" "${OpenAL_SOURCE_DIR}/common" "${OpenAL_BINARY_DIR}")

if (NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE RelWithDebInfo CACHE STRING
        "Choose the type of build, options are: Debug Release RelWithDebInfo MinSizeRel."
        FORCE)
endif ()
if (NOT CMAKE_DEBUG_POSTFIX)
    set(CMAKE_DEBUG_POSTFIX "" CACHE STRING
        "Library postfix for debug builds. Normally left blank."
        FORCE)
endif ()

if (MSVC)
    set(CPP_DEFS ${CPP_DEFS} _CRT_SECURE_NO_WARNINGS _CRT_NONSTDC_NO_DEPRECATE)
    set(C_FLAGS ${C_FLAGS} /wd4098)

    if (NOT DXSDK_DIR)
        string(REGEX REPLACE "\\\\" "/" DXSDK_DIR "$ENV{DXSDK_DIR}")
    else ()
        string(REGEX REPLACE "\\\\" "/" DXSDK_DIR "${DXSDK_DIR}")
    endif ()
    if (DXSDK_DIR)
        message(STATUS "Using DirectX SDK directory: ${DXSDK_DIR}")
    endif ()

    option(FORCE_STATIC_VCRT "Force /MT for static VC runtimes" OFF)
    if (FORCE_STATIC_VCRT)
        foreach (flag_var
                CMAKE_C_FLAGS CMAKE_C_FLAGS_DEBUG CMAKE_C_FLAGS_RELEASE
                CMAKE_C_FLAGS_MINSIZEREL CMAKE_C_FLAGS_RELWITHDEBINFO)
            if (${flag_var} MATCHES "/MD")
                string(REGEX REPLACE "/MD" "/MT" ${flag_var} "${${flag_var}}")
            endif ()
        endforeach (flag_var)
    endif ()
else ()
    set(C_FLAGS ${C_FLAGS} -Winline -Wall)
    check_c_compiler_flag(-Wextra HAVE_W_EXTRA)
    if (HAVE_W_EXTRA)
        set(C_FLAGS ${C_FLAGS} -Wextra)
    endif ()

    if (ALSOFT_WERROR)
        set(C_FLAGS ${C_FLAGS} -Werror)
    endif ()

    # We want RelWithDebInfo to actually include debug stuff (define _DEBUG
    # instead of NDEBUG)
    foreach (flag_var  CMAKE_C_FLAGS_RELWITHDEBINFO CMAKE_CXX_FLAGS_RELWITHDEBINFO)
        if (${flag_var} MATCHES "-DNDEBUG")
            string(REGEX REPLACE "-DNDEBUG" "-D_DEBUG" ${flag_var} "${${flag_var}}")
        endif ()
    endforeach ()

    check_c_compiler_flag(-fno-math-errno HAVE_FNO_MATH_ERRNO)
    if (HAVE_FNO_MATH_ERRNO)
        set(C_FLAGS ${C_FLAGS} -fno-math-errno)
    endif ()

    check_c_source_compiles("int foo() __attribute__((destructor));
                             int main() {return 0;}" HAVE_GCC_DESTRUCTOR)

    option(ALSOFT_STATIC_LIBGCC "Force -static-libgcc for static GCC runtimes" OFF)
    if (ALSOFT_STATIC_LIBGCC)
        set(OLD_REQUIRED_LIBRARIES ${CMAKE_REQUIRED_LIBRARIES})
        set(CMAKE_REQUIRED_LIBRARIES ${CMAKE_REQUIRED_LIBRARIES} -static-libgcc)
        check_c_source_compiles(
"#include <stdlib.h>
int main()
{
    return 0;
}"
            HAVE_STATIC_LIBGCC_SWITCH
        )
        if (HAVE_STATIC_LIBGCC_SWITCH)
            set(LINKER_FLAGS ${LINKER_FLAGS} -static-libgcc)
        endif ()
        set(CMAKE_REQUIRED_LIBRARIES ${OLD_REQUIRED_LIBRARIES})
        unset(OLD_REQUIRED_LIBRARIES)
    endif ()
endif ()

# Set visibility/export options if available
if (WIN32)
    set(EXPORT_DECL "__declspec(dllexport)")
    if (NOT MINGW)
        set(ALIGN_DECL "__declspec(align(x))")
    else ()
        set(ALIGN_DECL "__declspec(aligned(x))")
    endif ()
else ()
    set(OLD_REQUIRED_FLAGS "${CMAKE_REQUIRED_FLAGS}")
    # Yes GCC, really don't accept visibility modes you don't support
    set(CMAKE_REQUIRED_FLAGS "${OLD_REQUIRED_FLAGS} -Wattributes -Werror")

    check_c_source_compiles("int foo() __attribute__((visibility(\"protected\")));
                             int main() {return 0;}" HAVE_GCC_PROTECTED_VISIBILITY)
    if (HAVE_GCC_PROTECTED_VISIBILITY)
        set(EXPORT_DECL "__attribute__((visibility(\"protected\")))")
    else ()
        check_c_source_compiles("int foo() __attribute__((visibility(\"default\")));
                                 int main() {return 0;}" HAVE_GCC_DEFAULT_VISIBILITY)
        if (HAVE_GCC_DEFAULT_VISIBILITY)
            set(EXPORT_DECL "__attribute__((visibility(\"default\")))")
        endif ()
    endif ()

    if (HAVE_GCC_PROTECTED_VISIBILITY OR HAVE_GCC_DEFAULT_VISIBILITY)
        check_c_compiler_flag(-fvisibility=hidden HAVE_VISIBILITY_HIDDEN_SWITCH)
        if (HAVE_VISIBILITY_HIDDEN_SWITCH)
            set(C_FLAGS ${C_FLAGS} -fvisibility=hidden)
        endif ()
    endif ()

    check_c_source_compiles("int foo __attribute__((aligned(16)));
                             int main() {return 0;}" HAVE_ATTRIBUTE_ALIGNED)
    if (HAVE_ATTRIBUTE_ALIGNED)
        set(ALIGN_DECL "__attribute__((aligned(x)))")
    endif ()

    set(CMAKE_REQUIRED_FLAGS "${OLD_REQUIRED_FLAGS}")
endif ()

check_c_source_compiles("
int main()
{
    float *ptr;
    ptr = __builtin_assume_aligned(ptr, 16);
    return 0;
}" HAVE___BUILTIN_ASSUME_ALIGNED)
if (HAVE___BUILTIN_ASSUME_ALIGNED)
    set(ASSUME_ALIGNED_DECL "__builtin_assume_aligned(x, y)")
else ()
    set(ASSUME_ALIGNED_DECL "(x)")
endif ()

set(SSE_SWITCH "")
set(SSE2_SWITCH "")
set(SSE3_SWITCH "")
set(SSE4_1_SWITCH "")
set(FPU_NEON_SWITCH "")

check_c_compiler_flag(-msse HAVE_MSSE_SWITCH)
if (HAVE_MSSE_SWITCH)
    set(SSE_SWITCH "-msse")
endif ()
check_c_compiler_flag(-msse2 HAVE_MSSE2_SWITCH)
if (HAVE_MSSE2_SWITCH)
    set(SSE2_SWITCH "-msse2")
endif ()
check_c_compiler_flag(-msse3 HAVE_MSSE3_SWITCH)
if (HAVE_MSSE3_SWITCH)
    set(SSE3_SWITCH "-msse3")
endif ()
check_c_compiler_flag(-msse4.1 HAVE_MSSE4_1_SWITCH)
if (HAVE_MSSE4_1_SWITCH)
    set(SSE4_1_SWITCH "-msse4.1")
endif ()
check_c_compiler_flag(-mfpu=neon HAVE_MFPU_NEON_SWITCH)
if (HAVE_MFPU_NEON_SWITCH)
    set(FPU_NEON_SWITCH "-mfpu=neon")
endif ()

set(FPMATH_SET "0")
if (CMAKE_SIZEOF_VOID_P MATCHES "4")
    if (SSE_SWITCH OR MSVC)
        option(ALSOFT_ENABLE_SSE_CODEGEN "Enable SSE code generation instead of x87 for 32-bit targets." TRUE)
    endif ()
    if (SSE2_SWITCH OR MSVC)
        option(ALSOFT_ENABLE_SSE2_CODEGEN "Enable SSE2 code generation instead of x87 for 32-bit targets." TRUE)
    endif ()

    if (ALSOFT_ENABLE_SSE2_CODEGEN)
        if (SSE2_SWITCH)
            check_c_compiler_flag("${SSE2_SWITCH} -mfpmath=sse" HAVE_MFPMATH_SSE_2)
            if (HAVE_MFPMATH_SSE_2)
                set(C_FLAGS ${C_FLAGS} ${SSE2_SWITCH} -mfpmath=sse)
                set(FPMATH_SET 2)
            endif ()
        ELSEIF(MSVC)
            check_c_compiler_flag("/arch:SSE2" HAVE_ARCH_SSE2)
            if (HAVE_ARCH_SSE2)
                set(C_FLAGS ${C_FLAGS} "/arch:SSE2")
                set(FPMATH_SET 2)
            endif ()
        endif ()
    endif ()
    if (ALSOFT_ENABLE_SSE_CODEGEN AND NOT FPMATH_SET)
        if (SSE_SWITCH)
            check_c_compiler_flag("${SSE_SWITCH} -mfpmath=sse" HAVE_MFPMATH_SSE)
            if (HAVE_MFPMATH_SSE)
                set(C_FLAGS ${C_FLAGS} ${SSE_SWITCH} -mfpmath=sse)
                set(FPMATH_SET 1)
            endif ()
        ELSEIF(MSVC)
            check_c_compiler_flag("/arch:SSE" HAVE_ARCH_SSE)
            if (HAVE_ARCH_SSE)
                set(C_FLAGS ${C_FLAGS} "/arch:SSE")
                set(FPMATH_SET 1)
            endif ()
        endif ()
    endif ()
endif ()


check_c_source_compiles("int foo(const char *str, ...) __attribute__((format(printf, 1, 2)));
                         int main() {return 0;}" HAVE_GCC_FORMAT)

check_include_file(stdbool.h HAVE_STDBOOL_H)
check_include_file(stdalign.h HAVE_STDALIGN_H)
check_include_file(malloc.h HAVE_MALLOC_H)
check_include_file(dirent.h HAVE_DIRENT_H)
check_include_file(strings.h HAVE_STRINGS_H)
check_include_file(cpuid.h HAVE_CPUID_H)
check_include_file(intrin.h HAVE_INTRIN_H)
check_include_file(sys/sysconf.h HAVE_SYS_SYSCONF_H)
check_include_file(fenv.h HAVE_FENV_H)
check_include_file(float.h HAVE_FLOAT_H)
check_include_file(ieeefp.h HAVE_IEEEFP_H)
check_include_file(guiddef.h HAVE_GUIDDEF_H)
if (NOT HAVE_GUIDDEF_H)
    check_include_file(initguid.h HAVE_INITGUID_H)
endif ()

# Some systems need libm for some of the following math functions to work
set(MATH_LIB )
check_library_exists(m pow "" HAVE_LIBM)
if (HAVE_LIBM)
    set(MATH_LIB ${MATH_LIB} m)
    set(CMAKE_REQUIRED_LIBRARIES ${CMAKE_REQUIRED_LIBRARIES} m)
endif ()

# Check for the dlopen API (for dynamicly loading backend libs)
if (ALSOFT_DLOPEN)
    check_library_exists(dl dlopen "" HAVE_LIBDL)
    if (HAVE_LIBDL)
        set(EXTRA_LIBS dl ${EXTRA_LIBS})
        set(CMAKE_REQUIRED_LIBRARIES ${CMAKE_REQUIRED_LIBRARIES} dl)
    endif ()

    check_include_file(dlfcn.h HAVE_DLFCN_H)
endif ()

# Check for a cpuid intrinsic
if (HAVE_CPUID_H)
    check_c_source_compiles("#include <cpuid.h>
        int main()
        {
            unsigned int eax, ebx, ecx, edx;
            return __get_cpuid(0, &eax, &ebx, &ecx, &edx);
        }" HAVE_GCC_GET_CPUID)
endif ()
if (HAVE_INTRIN_H)
    check_c_source_compiles("#include <intrin.h>
        int main()
        {
            int regs[4];
            __cpuid(regs, 0);
            return regs[0];
        }" HAVE_CPUID_INTRINSIC)
    check_c_source_compiles("#include <intrin.h>
        int main()
        {
            unsigned long idx = 0;
            _BitScanForward64(&idx, 1);
            return idx;
        }" HAVE_BITSCANFORWARD64_INTRINSIC)
    check_c_source_compiles("#include <intrin.h>
        int main()
        {
            unsigned long idx = 0;
            _BitScanForward(&idx, 1);
            return idx;
        }" HAVE_BITSCANFORWARD_INTRINSIC)
endif ()

check_symbol_exists(sysconf          unistd.h HAVE_SYSCONF)
check_symbol_exists(aligned_alloc    stdlib.h HAVE_ALIGNED_ALLOC)
check_symbol_exists(posix_memalign   stdlib.h HAVE_POSIX_MEMALIGN)
check_symbol_exists(_aligned_malloc  malloc.h HAVE__ALIGNED_MALLOC)
check_symbol_exists(proc_pidpath     libproc.h HAVE_PROC_PIDPATH)
check_symbol_exists(lrintf math.h HAVE_LRINTF)
check_symbol_exists(modff  math.h HAVE_MODFF)
check_symbol_exists(log2f  math.h HAVE_LOG2F)
check_symbol_exists(cbrtf  math.h HAVE_CBRTF)
check_symbol_exists(copysignf math.h HAVE_COPYSIGNF)

if (HAVE_FLOAT_H)
    check_symbol_exists(_controlfp float.h HAVE__CONTROLFP)
    check_symbol_exists(__control87_2 float.h HAVE___CONTROL87_2)
endif ()

check_function_exists(stat HAVE_STAT)
check_function_exists(strtof HAVE_STRTOF)
check_function_exists(strcasecmp HAVE_STRCASECMP)
if (NOT HAVE_STRCASECMP)
    check_function_exists(_stricmp HAVE__STRICMP)
    if (NOT HAVE__STRICMP)
        message(FATAL_ERROR "No case-insensitive compare function found, please report!")
    endif ()

    set(CPP_DEFS ${CPP_DEFS} strcasecmp=_stricmp)
endif ()

check_function_exists(strncasecmp HAVE_STRNCASECMP)
if (NOT HAVE_STRNCASECMP)
    check_function_exists(_strnicmp HAVE__STRNICMP)
    if (NOT HAVE__STRNICMP)
        message(FATAL_ERROR "No case-insensitive size-limitted compare function found, please report!")
    endif ()

    set(CPP_DEFS ${CPP_DEFS} strncasecmp=_strnicmp)
endif ()

check_symbol_exists(strnlen string.h HAVE_STRNLEN)
check_symbol_exists(snprintf stdio.h HAVE_SNPRINTF)
if (NOT HAVE_SNPRINTF)
    check_function_exists(_snprintf HAVE__SNPRINTF)
    if (NOT HAVE__SNPRINTF)
        message(FATAL_ERROR "No snprintf function found, please report!")
    endif ()

    set(CPP_DEFS ${CPP_DEFS} snprintf=_snprintf)
endif ()

check_symbol_exists(isfinite math.h HAVE_ISFINITE)
if (NOT HAVE_ISFINITE)
    check_function_exists(finite HAVE_FINITE)
    if (NOT HAVE_FINITE)
        check_function_exists(_finite HAVE__FINITE)
        if (NOT HAVE__FINITE)
            message(FATAL_ERROR "No isfinite function found, please report!")
        endif ()
        set(CPP_DEFS ${CPP_DEFS} isfinite=_finite)
    else ()
        set(CPP_DEFS ${CPP_DEFS} isfinite=finite)
    endif ()
endif ()

check_symbol_exists(isnan math.h HAVE_ISNAN)
if (NOT HAVE_ISNAN)
    check_function_exists(_isnan HAVE__ISNAN)
    if (NOT HAVE__ISNAN)
        message(FATAL_ERROR "No isnan function found, please report!")
    endif ()

    set(CPP_DEFS ${CPP_DEFS} isnan=_isnan)
endif ()


# Check if we have Windows headers
set(OLD_REQUIRED_DEFINITIONS ${CMAKE_REQUIRED_DEFINITIONS})
set(CMAKE_REQUIRED_DEFINITIONS ${CMAKE_REQUIRED_DEFINITIONS} -D_WIN32_WINNT=0x0502)
check_include_file(windows.h HAVE_WINDOWS_H)
set(CMAKE_REQUIRED_DEFINITIONS ${OLD_REQUIRED_DEFINITIONS})
unset(OLD_REQUIRED_DEFINITIONS)

if (NOT HAVE_WINDOWS_H)
    check_symbol_exists(gettimeofday sys/time.h HAVE_GETTIMEOFDAY)
    if (NOT HAVE_GETTIMEOFDAY)
        message(FATAL_ERROR "No timing function found!")
    endif ()

    check_symbol_exists(nanosleep time.h HAVE_NANOSLEEP)
    if (NOT HAVE_NANOSLEEP)
        message(FATAL_ERROR "No sleep function found!")
    endif ()

    # We need pthreads outside of Windows
    check_include_file(pthread.h HAVE_PTHREAD_H)
    if (NOT HAVE_PTHREAD_H)
        message(FATAL_ERROR "PThreads is required for non-Windows builds!")
    endif ()
    # Some systems need pthread_np.h to get recursive mutexes
    check_include_files("pthread.h;pthread_np.h" HAVE_PTHREAD_NP_H)

    check_c_compiler_flag(-pthread HAVE_PTHREAD)
    if (HAVE_PTHREAD)
        set(CMAKE_REQUIRED_FLAGS "${CMAKE_REQUIRED_FLAGS} -pthread")
        set(C_FLAGS ${C_FLAGS} -pthread)
        set(LINKER_FLAGS ${LINKER_FLAGS} -pthread)
    endif ()

    check_library_exists(pthread pthread_create "" HAVE_LIBPTHREAD)
    if (HAVE_LIBPTHREAD)
        set(EXTRA_LIBS pthread ${EXTRA_LIBS})
    endif ()

    check_symbol_exists(pthread_setschedparam pthread.h HAVE_PTHREAD_SETSCHEDPARAM)

    if (HAVE_PTHREAD_NP_H)
        check_symbol_exists(pthread_setname_np "pthread.h;pthread_np.h" HAVE_PTHREAD_SETNAME_NP)
        if (NOT HAVE_PTHREAD_SETNAME_NP)
            check_symbol_exists(pthread_set_name_np "pthread.h;pthread_np.h" HAVE_PTHREAD_SET_NAME_NP)
        else ()
            check_c_source_compiles("
#include <pthread.h>
#include <pthread_np.h>
int main()
{
    pthread_setname_np(\"testname\");
    return 0;
}"
                PTHREAD_SETNAME_NP_ONE_PARAM
            )
            check_c_source_compiles("
#include <pthread.h>
#include <pthread_np.h>
int main()
{
    pthread_setname_np(pthread_self(), \"%s\", \"testname\");
    return 0;
}"
                PTHREAD_SETNAME_NP_THREE_PARAMS
            )
        endif ()
        check_symbol_exists(pthread_mutexattr_setkind_np "pthread.h;pthread_np.h" HAVE_PTHREAD_MUTEXATTR_SETKIND_NP)
    else ()
        check_symbol_exists(pthread_setname_np pthread.h HAVE_PTHREAD_SETNAME_NP)
        if (NOT HAVE_PTHREAD_SETNAME_NP)
            check_symbol_exists(pthread_set_name_np pthread.h HAVE_PTHREAD_SET_NAME_NP)
        else ()
            check_c_source_compiles("
#include <pthread.h>
int main()
{
    pthread_setname_np(\"testname\");
    return 0;
}"
                PTHREAD_SETNAME_NP_ONE_PARAM
            )
            check_c_source_compiles("
#include <pthread.h>
int main()
{
    pthread_setname_np(pthread_self(), \"%s\", \"testname\");
    return 0;
}"
                PTHREAD_SETNAME_NP_THREE_PARAMS
            )
        endif ()
        check_symbol_exists(pthread_mutexattr_setkind_np pthread.h HAVE_PTHREAD_MUTEXATTR_SETKIND_NP)
    endif ()

    check_symbol_exists(pthread_mutex_timedlock pthread.h HAVE_PTHREAD_MUTEX_TIMEDLOCK)

    check_library_exists(rt clock_gettime "" HAVE_LIBRT)
    if (HAVE_LIBRT)
        set(EXTRA_LIBS rt ${EXTRA_LIBS})
    endif ()
endif ()

check_symbol_exists(getopt unistd.h HAVE_GETOPT)

# Check for a 64-bit type
check_include_file(stdint.h HAVE_STDINT_H)
if (NOT HAVE_STDINT_H)
    if (HAVE_WINDOWS_H)
        check_c_source_compiles("#define _WIN32_WINNT 0x0502
                                 #include <windows.h>
                                 __int64 foo;
                                 int main() {return 0;}" HAVE___INT64)
    endif ()
    if (NOT HAVE___INT64)
        if (NOT SIZEOF_LONG MATCHES "8")
            if (NOT SIZEOF_LONG_LONG MATCHES "8")
                message(FATAL_ERROR "No 64-bit types found, please report!")
            endif ()
        endif ()
    endif ()
endif ()


set(COMMON_OBJS
    common/alcomplex.c
    common/alcomplex.h
    common/align.h
    common/almalloc.c
    common/almalloc.h
    common/atomic.c
    common/atomic.h
    common/bool.h
    common/math_defs.h
    common/rwlock.c
    common/rwlock.h
    common/static_assert.h
    common/threads.c
    common/threads.h
    common/uintmap.c
    common/uintmap.h
)
set(OPENAL_OBJS
    OpenAL32/Include/bs2b.h
    OpenAL32/Include/alMain.h
    OpenAL32/Include/alu.h

    OpenAL32/Include/alAuxEffectSlot.h
    OpenAL32/alAuxEffectSlot.c
    OpenAL32/Include/alBuffer.h
    OpenAL32/alBuffer.c
    OpenAL32/Include/alEffect.h
    OpenAL32/alEffect.c
    OpenAL32/Include/alError.h
    OpenAL32/alError.c
    OpenAL32/alExtension.c
    OpenAL32/Include/alFilter.h
    OpenAL32/alFilter.c
    OpenAL32/Include/alListener.h
    OpenAL32/alListener.c
    OpenAL32/Include/alSource.h
    OpenAL32/alSource.c
    OpenAL32/alState.c
    OpenAL32/event.c
    OpenAL32/Include/sample_cvt.h
    OpenAL32/sample_cvt.c
)
set(ALC_OBJS
    Alc/ALc.c
    Alc/ALu.c
    Alc/alconfig.c
    Alc/alconfig.h
    Alc/bs2b.c
    Alc/converter.c
    Alc/converter.h
    Alc/inprogext.h
    Alc/mastering.c
    Alc/mastering.h
    Alc/ringbuffer.c
    Alc/ringbuffer.h
    Alc/effects/autowah.c
    Alc/effects/chorus.c
    Alc/effects/compressor.c
    Alc/effects/dedicated.c
    Alc/effects/distortion.c
    Alc/effects/echo.c
    Alc/effects/equalizer.c
    Alc/effects/fshifter.c
    Alc/effects/modulator.c
    Alc/effects/null.c
    Alc/effects/pshifter.c
    Alc/effects/reverb.c
    Alc/filters/defs.h
    Alc/filters/filter.c
    Alc/filters/nfc.c
    Alc/filters/nfc.h
    Alc/filters/splitter.c
    Alc/filters/splitter.h
    Alc/helpers.c
    Alc/alstring.h
    Alc/compat.h
    Alc/cpu_caps.h
    Alc/fpu_modes.h
    Alc/logging.h
    Alc/vector.h
    Alc/hrtf.c
    Alc/hrtf.h
    Alc/uhjfilter.c
    Alc/uhjfilter.h
    Alc/ambdec.c
    Alc/ambdec.h
    Alc/bformatdec.c
    Alc/bformatdec.h
    Alc/panning.c
    Alc/polymorphism.h
    Alc/mixvoice.c
    Alc/mixer/defs.h
    Alc/mixer/mixer_c.c
)


set(CPU_EXTS "Default")
set(HAVE_SSE        0)
set(HAVE_SSE2       0)
set(HAVE_SSE3       0)
set(HAVE_SSE4_1     0)
set(HAVE_NEON       0)

set(HAVE_ALSA       0)
set(HAVE_OSS        0)
set(HAVE_SOLARIS    0)
set(HAVE_SNDIO      0)
set(HAVE_QSA        0)
set(HAVE_DSOUND     0)
set(HAVE_WASAPI     0)
set(HAVE_WINMM      0)
set(HAVE_PORTAUDIO  0)
set(HAVE_PULSEAUDIO 0)
set(HAVE_COREAUDIO  0)
set(HAVE_OPENSL     0)
set(HAVE_WAVE       0)
set(HAVE_SDL2       0)

# Check for SSE support
option(ALSOFT_REQUIRE_SSE "Require SSE support" OFF)
check_include_file(xmmintrin.h HAVE_XMMINTRIN_H "${SSE_SWITCH}")
if (HAVE_XMMINTRIN_H)
    option(ALSOFT_CPUEXT_SSE "Enable SSE support" ON)
    if (ALSOFT_CPUEXT_SSE)
        if (ALIGN_DECL OR HAVE_C11_ALIGNAS)
            set(HAVE_SSE 1)
            set(ALC_OBJS  ${ALC_OBJS} Alc/mixer/mixer_sse.c)
            if (SSE_SWITCH)
                SET_SOURCE_FILES_PROPERTIES(Alc/mixer/mixer_sse.c PROPERTIES
                                            COMPILE_FLAGS "${SSE_SWITCH}")
            endif ()
            set(CPU_EXTS "${CPU_EXTS}, SSE")
        endif ()
    endif ()
endif ()
if (ALSOFT_REQUIRE_SSE AND NOT HAVE_SSE)
    message(FATAL_ERROR "Failed to enabled required SSE CPU extensions")
endif ()

option(ALSOFT_REQUIRE_SSE2 "Require SSE2 support" OFF)
check_include_file(emmintrin.h HAVE_EMMINTRIN_H "${SSE2_SWITCH}")
if (HAVE_EMMINTRIN_H)
    option(ALSOFT_CPUEXT_SSE2 "Enable SSE2 support" ON)
    if (HAVE_SSE AND ALSOFT_CPUEXT_SSE2)
        if (ALIGN_DECL OR HAVE_C11_ALIGNAS)
            set(HAVE_SSE2 1)
            set(ALC_OBJS  ${ALC_OBJS} Alc/mixer/mixer_sse2.c)
            if (SSE2_SWITCH)
                SET_SOURCE_FILES_PROPERTIES(Alc/mixer/mixer_sse2.c PROPERTIES
                                            COMPILE_FLAGS "${SSE2_SWITCH}")
            endif ()
            set(CPU_EXTS "${CPU_EXTS}, SSE2")
        endif ()
    endif ()
endif ()
if (ALSOFT_REQUIRE_SSE2 AND NOT HAVE_SSE2)
    message(FATAL_ERROR "Failed to enable required SSE2 CPU extensions")
endif ()

option(ALSOFT_REQUIRE_SSE3 "Require SSE3 support" OFF)
check_include_file(pmmintrin.h HAVE_PMMINTRIN_H "${SSE3_SWITCH}")
if (HAVE_EMMINTRIN_H)
    option(ALSOFT_CPUEXT_SSE3 "Enable SSE3 support" ON)
    if (HAVE_SSE2 AND ALSOFT_CPUEXT_SSE3)
        if (ALIGN_DECL OR HAVE_C11_ALIGNAS)
            set(HAVE_SSE3 1)
            set(ALC_OBJS  ${ALC_OBJS} Alc/mixer/mixer_sse3.c)
            if (SSE2_SWITCH)
                SET_SOURCE_FILES_PROPERTIES(Alc/mixer/mixer_sse3.c PROPERTIES
                                            COMPILE_FLAGS "${SSE3_SWITCH}")
            endif ()
            set(CPU_EXTS "${CPU_EXTS}, SSE3")
        endif ()
    endif ()
endif ()
if (ALSOFT_REQUIRE_SSE3 AND NOT HAVE_SSE3)
    message(FATAL_ERROR "Failed to enable required SSE3 CPU extensions")
endif ()

option(ALSOFT_REQUIRE_SSE4_1 "Require SSE4.1 support" OFF)
check_include_file(smmintrin.h HAVE_SMMINTRIN_H "${SSE4_1_SWITCH}")
if (HAVE_SMMINTRIN_H)
    option(ALSOFT_CPUEXT_SSE4_1 "Enable SSE4.1 support" ON)
    if (HAVE_SSE2 AND ALSOFT_CPUEXT_SSE4_1)
        if (ALIGN_DECL OR HAVE_C11_ALIGNAS)
            set(HAVE_SSE4_1 1)
            set(ALC_OBJS  ${ALC_OBJS} Alc/mixer/mixer_sse41.c)
            if (SSE4_1_SWITCH)
                SET_SOURCE_FILES_PROPERTIES(Alc/mixer/mixer_sse41.c PROPERTIES
                                            COMPILE_FLAGS "${SSE4_1_SWITCH}")
            endif ()
            set(CPU_EXTS "${CPU_EXTS}, SSE4.1")
        endif ()
    endif ()
endif ()
if (ALSOFT_REQUIRE_SSE4_1 AND NOT HAVE_SSE4_1)
    message(FATAL_ERROR "Failed to enable required SSE4.1 CPU extensions")
endif ()

# Check for ARM Neon support
option(ALSOFT_REQUIRE_NEON "Require ARM Neon support" OFF)
check_include_file(arm_neon.h HAVE_ARM_NEON_H ${FPU_NEON_SWITCH})
if (HAVE_ARM_NEON_H)
    option(ALSOFT_CPUEXT_NEON "Enable ARM Neon support" ON)
    if (ALSOFT_CPUEXT_NEON)
        set(HAVE_NEON 1)
        set(ALC_OBJS  ${ALC_OBJS} Alc/mixer/mixer_neon.c)
        if (FPU_NEON_SWITCH)
            SET_SOURCE_FILES_PROPERTIES(Alc/mixer/mixer_neon.c PROPERTIES
                                        COMPILE_FLAGS "${FPU_NEON_SWITCH}")
        endif ()
        set(CPU_EXTS "${CPU_EXTS}, Neon")
    endif ()
endif ()
if (ALSOFT_REQUIRE_NEON AND NOT HAVE_NEON)
    message(FATAL_ERROR "Failed to enabled required ARM Neon CPU extensions")
endif ()


if (WIN32 OR HAVE_DLFCN_H)
    set(IS_LINKED "")
    MACRO(ADD_BACKEND_LIBS _LIBS)
    ENDMACRO()
else ()
    set(IS_LINKED " (linked)")
    MACRO(ADD_BACKEND_LIBS _LIBS)
        set(EXTRA_LIBS ${_LIBS} ${EXTRA_LIBS})
    ENDMACRO()
endif ()

set(BACKENDS "")
set(ALC_OBJS  ${ALC_OBJS}
    Alc/backends/base.c
    Alc/backends/base.h
    # Default backends, always available
    Alc/backends/loopback.c
    Alc/backends/null.c
)

# Check ALSA backend
option(ALSOFT_REQUIRE_ALSA "Require ALSA backend" OFF)
find_package(ALSA)
if (ALSA_FOUND)
    option(ALSOFT_BACKEND_ALSA "Enable ALSA backend" ON)
    if (ALSOFT_BACKEND_ALSA)
        set(HAVE_ALSA 1)
        set(BACKENDS  "${BACKENDS} ALSA${IS_LINKED},")
        set(ALC_OBJS  ${ALC_OBJS} Alc/backends/alsa.c)
        ADD_BACKEND_LIBS(${ALSA_LIBRARIES})
        set(INC_PATHS ${INC_PATHS} ${ALSA_INCLUDE_DIRS})
    endif ()
endif ()
if (ALSOFT_REQUIRE_ALSA AND NOT HAVE_ALSA)
    message(FATAL_ERROR "Failed to enabled required ALSA backend")
endif ()

# Check OSS backend
option(ALSOFT_REQUIRE_OSS "Require OSS backend" OFF)
find_package(OSS)
if (OSS_FOUND)
    option(ALSOFT_BACKEND_OSS "Enable OSS backend" ON)
    if (ALSOFT_BACKEND_OSS)
        set(HAVE_OSS 1)
        set(BACKENDS  "${BACKENDS} OSS,")
        set(ALC_OBJS  ${ALC_OBJS} Alc/backends/oss.c)
        if (OSS_LIBRARIES)
            set(EXTRA_LIBS ${OSS_LIBRARIES} ${EXTRA_LIBS})
        endif ()
        set(INC_PATHS ${INC_PATHS} ${OSS_INCLUDE_DIRS})
    endif ()
endif ()
if (ALSOFT_REQUIRE_OSS AND NOT HAVE_OSS)
    message(FATAL_ERROR "Failed to enabled required OSS backend")
endif ()

# Check Solaris backend
option(ALSOFT_REQUIRE_SOLARIS "Require Solaris backend" OFF)
find_package(AudioIO)
if (AUDIOIO_FOUND)
    option(ALSOFT_BACKEND_SOLARIS "Enable Solaris backend" ON)
    if (ALSOFT_BACKEND_SOLARIS)
        set(HAVE_SOLARIS 1)
        set(BACKENDS  "${BACKENDS} Solaris,")
        set(ALC_OBJS  ${ALC_OBJS} Alc/backends/solaris.c)
        set(INC_PATHS ${INC_PATHS} ${AUDIOIO_INCLUDE_DIRS})
    endif ()
endif ()
if (ALSOFT_REQUIRE_SOLARIS AND NOT HAVE_SOLARIS)
    message(FATAL_ERROR "Failed to enabled required Solaris backend")
endif ()

# Check SndIO backend
option(ALSOFT_REQUIRE_SNDIO "Require SndIO backend" OFF)
find_package(SoundIO)
if (SOUNDIO_FOUND)
    option(ALSOFT_BACKEND_SNDIO "Enable SndIO backend" ON)
    if (ALSOFT_BACKEND_SNDIO)
        set(HAVE_SNDIO 1)
        set(BACKENDS  "${BACKENDS} SndIO (linked),")
        set(ALC_OBJS  ${ALC_OBJS} Alc/backends/sndio.c)
        set(EXTRA_LIBS ${SOUNDIO_LIBRARIES} ${EXTRA_LIBS})
        set(INC_PATHS ${INC_PATHS} ${SOUNDIO_INCLUDE_DIRS})
    endif ()
endif ()
if (ALSOFT_REQUIRE_SNDIO AND NOT HAVE_SNDIO)
    message(FATAL_ERROR "Failed to enabled required SndIO backend")
endif ()

# Check QSA backend
option(ALSOFT_REQUIRE_QSA "Require QSA backend" OFF)
find_package(QSA)
if (QSA_FOUND)
    option(ALSOFT_BACKEND_QSA "Enable QSA backend" ON)
    if (ALSOFT_BACKEND_QSA)
        set(HAVE_QSA 1)
        set(BACKENDS  "${BACKENDS} QSA (linked),")
        set(ALC_OBJS  ${ALC_OBJS} Alc/backends/qsa.c)
        set(EXTRA_LIBS ${QSA_LIBRARIES} ${EXTRA_LIBS})
        set(INC_PATHS ${INC_PATHS} ${QSA_INCLUDE_DIRS})
    endif ()
endif ()
if (ALSOFT_REQUIRE_QSA AND NOT HAVE_QSA)
    message(FATAL_ERROR "Failed to enabled required QSA backend")
endif ()

# Check Windows-only backends
option(ALSOFT_REQUIRE_WINMM "Require Windows Multimedia backend" OFF)
option(ALSOFT_REQUIRE_DSOUND "Require DirectSound backend" OFF)
option(ALSOFT_REQUIRE_WASAPI "Require WASAPI backend" OFF)
if (HAVE_WINDOWS_H)
    set(OLD_REQUIRED_DEFINITIONS ${CMAKE_REQUIRED_DEFINITIONS})
    set(CMAKE_REQUIRED_DEFINITIONS ${CMAKE_REQUIRED_DEFINITIONS} -D_WIN32_WINNT=0x0502)

    # Check MMSystem backend
    check_include_files("windows.h;mmsystem.h" HAVE_MMSYSTEM_H)
    if (HAVE_MMSYSTEM_H)
        check_shared_function_exists(waveOutOpen "windows.h;mmsystem.h" winmm "" HAVE_LIBWINMM)
        if (HAVE_LIBWINMM)
            option(ALSOFT_BACKEND_WINMM "Enable Windows Multimedia backend" ON)
            if (ALSOFT_BACKEND_WINMM)
                set(HAVE_WINMM 1)
                set(BACKENDS  "${BACKENDS} WinMM,")
                set(ALC_OBJS  ${ALC_OBJS} Alc/backends/winmm.c)
                set(EXTRA_LIBS winmm ${EXTRA_LIBS})
            endif ()
        endif ()
    endif ()

    # Check DSound backend
    find_package(DSound)
    if (DSOUND_FOUND)
        option(ALSOFT_BACKEND_DSOUND "Enable DirectSound backend" ON)
        if (ALSOFT_BACKEND_DSOUND)
            set(HAVE_DSOUND 1)
            set(BACKENDS  "${BACKENDS} DirectSound${IS_LINKED},")
            set(ALC_OBJS  ${ALC_OBJS} Alc/backends/dsound.c)
            ADD_BACKEND_LIBS(${DSOUND_LIBRARIES})
            set(INC_PATHS ${INC_PATHS} ${DSOUND_INCLUDE_DIRS})
        endif ()
    endif ()

    # Check for WASAPI backend
    check_include_file(mmdeviceapi.h HAVE_MMDEVICEAPI_H)
    if (HAVE_MMDEVICEAPI_H)
        option(ALSOFT_BACKEND_WASAPI "Enable WASAPI backend" ON)
        if (ALSOFT_BACKEND_WASAPI)
            set(HAVE_WASAPI 1)
            set(BACKENDS  "${BACKENDS} WASAPI,")
            set(ALC_OBJS  ${ALC_OBJS} Alc/backends/wasapi.c)
        endif ()
    endif ()

    set(CMAKE_REQUIRED_DEFINITIONS ${OLD_REQUIRED_DEFINITIONS})
    unset(OLD_REQUIRED_DEFINITIONS)
endif ()
if (ALSOFT_REQUIRE_WINMM AND NOT HAVE_WINMM)
    message(FATAL_ERROR "Failed to enabled required WinMM backend")
endif ()
if (ALSOFT_REQUIRE_DSOUND AND NOT HAVE_DSOUND)
    message(FATAL_ERROR "Failed to enabled required DSound backend")
endif ()
if (ALSOFT_REQUIRE_WASAPI AND NOT HAVE_WASAPI)
    message(FATAL_ERROR "Failed to enabled required WASAPI backend")
endif ()

# Check PortAudio backend
option(ALSOFT_REQUIRE_PORTAUDIO "Require PortAudio backend" OFF)
find_package(PortAudio)
if (PORTAUDIO_FOUND)
    option(ALSOFT_BACKEND_PORTAUDIO "Enable PortAudio backend" ON)
    if (ALSOFT_BACKEND_PORTAUDIO)
        set(HAVE_PORTAUDIO 1)
        set(BACKENDS  "${BACKENDS} PortAudio${IS_LINKED},")
        set(ALC_OBJS  ${ALC_OBJS} Alc/backends/portaudio.c)
        ADD_BACKEND_LIBS(${PORTAUDIO_LIBRARIES})
        set(INC_PATHS ${INC_PATHS} ${PORTAUDIO_INCLUDE_DIRS})
    endif ()
endif ()
if (ALSOFT_REQUIRE_PORTAUDIO AND NOT HAVE_PORTAUDIO)
    message(FATAL_ERROR "Failed to enabled required PortAudio backend")
endif ()

# Check PulseAudio backend
option(ALSOFT_REQUIRE_PULSEAUDIO "Require PulseAudio backend" OFF)
find_package(PulseAudio)
if (PULSEAUDIO_FOUND)
    option(ALSOFT_BACKEND_PULSEAUDIO "Enable PulseAudio backend" ON)
    if (ALSOFT_BACKEND_PULSEAUDIO)
        set(HAVE_PULSEAUDIO 1)
        set(BACKENDS  "${BACKENDS} PulseAudio${IS_LINKED},")
        set(ALC_OBJS  ${ALC_OBJS} Alc/backends/pulseaudio.c)
        ADD_BACKEND_LIBS(${PULSEAUDIO_LIBRARIES})
        set(INC_PATHS ${INC_PATHS} ${PULSEAUDIO_INCLUDE_DIRS})
    endif ()
endif ()
if (ALSOFT_REQUIRE_PULSEAUDIO AND NOT HAVE_PULSEAUDIO)
    message(FATAL_ERROR "Failed to enabled required PulseAudio backend")
endif ()

# Check JACK backend
option(ALSOFT_REQUIRE_JACK "Require JACK backend" OFF)
find_package(JACK)
if (JACK_FOUND)
    option(ALSOFT_BACKEND_JACK "Enable JACK backend" ON)
    if (ALSOFT_BACKEND_JACK)
        set(HAVE_JACK 1)
        set(BACKENDS  "${BACKENDS} JACK${IS_LINKED},")
        set(ALC_OBJS  ${ALC_OBJS} Alc/backends/jack.c)
        ADD_BACKEND_LIBS(${JACK_LIBRARIES})
        set(INC_PATHS ${INC_PATHS} ${JACK_INCLUDE_DIRS})
    endif ()
endif ()
if (ALSOFT_REQUIRE_JACK AND NOT HAVE_JACK)
    message(FATAL_ERROR "Failed to enabled required JACK backend")
endif ()

# Check CoreAudio backend
option(ALSOFT_REQUIRE_COREAUDIO "Require CoreAudio backend" OFF)
find_library(COREAUDIO_FRAMEWORK
             NAMES CoreAudio
             PATHS /System/Library/Frameworks
)
if (COREAUDIO_FRAMEWORK)
    option(ALSOFT_BACKEND_COREAUDIO "Enable CoreAudio backend" ON)
    if (ALSOFT_BACKEND_COREAUDIO)
        set(HAVE_COREAUDIO 1)
        set(ALC_OBJS  ${ALC_OBJS} Alc/backends/coreaudio.c)
        set(BACKENDS  "${BACKENDS} CoreAudio,")
        set(EXTRA_LIBS ${COREAUDIO_FRAMEWORK} ${EXTRA_LIBS})
        set(EXTRA_LIBS /System/Library/Frameworks/AudioUnit.framework ${EXTRA_LIBS})
        set(EXTRA_LIBS /System/Library/Frameworks/ApplicationServices.framework ${EXTRA_LIBS})

        # Some versions of OSX may need the AudioToolbox framework. Add it if
        # it's found.
        find_library(AUDIOTOOLBOX_LIBRARY
                     NAMES AudioToolbox
                     PATHS ~/Library/Frameworks
                           /Library/Frameworks
                           /System/Library/Frameworks
                    )
        if (AUDIOTOOLBOX_LIBRARY)
            set(EXTRA_LIBS ${AUDIOTOOLBOX_LIBRARY} ${EXTRA_LIBS})
        endif ()
    endif ()
endif ()
if (ALSOFT_REQUIRE_COREAUDIO AND NOT HAVE_COREAUDIO)
    message(FATAL_ERROR "Failed to enabled required CoreAudio backend")
endif ()

# Check for OpenSL (Android) backend
option(ALSOFT_REQUIRE_OPENSL "Require OpenSL backend" OFF)
check_include_files("SLES/OpenSLES.h;SLES/OpenSLES_Android.h" HAVE_SLES_OPENSLES_ANDROID_H)
if (HAVE_SLES_OPENSLES_ANDROID_H)
    CHECK_SHARED_FUNCTION_EXISTS(slCreateEngine "SLES/OpenSLES.h" OpenSLES "" HAVE_LIBOPENSLES)
    if (HAVE_LIBOPENSLES)
        option(ALSOFT_BACKEND_OPENSL "Enable OpenSL backend" ON)
        if (ALSOFT_BACKEND_OPENSL)
            set(HAVE_OPENSL 1)
            set(ALC_OBJS  ${ALC_OBJS} Alc/backends/opensl.c)
            set(BACKENDS  "${BACKENDS} OpenSL,")
            set(EXTRA_LIBS OpenSLES ${EXTRA_LIBS})
        endif ()
    endif ()
endif ()
if (ALSOFT_REQUIRE_OPENSL AND NOT HAVE_OPENSL)
    message(FATAL_ERROR "Failed to enabled required OpenSL backend")
endif ()

# Check for SDL2 backend
option(ALSOFT_REQUIRE_SDL2 "Require SDL2 backend" OFF)
find_package(SDL2)
if (SDL2_FOUND)
    # Off by default, since it adds a runtime dependency
    option(ALSOFT_BACKEND_SDL2 "Enable SDL2 backend" OFF)
    if (ALSOFT_BACKEND_SDL2)
        set(HAVE_SDL2 1)
        set(ALC_OBJS  ${ALC_OBJS} Alc/backends/sdl2.c)
        set(BACKENDS  "${BACKENDS} SDL2,")
        set(EXTRA_LIBS ${SDL2_LIBRARY} ${EXTRA_LIBS})
        set(INC_PATHS ${INC_PATHS} ${SDL2_INCLUDE_DIR})
    endif ()
endif ()
if (ALSOFT_REQUIRE_SDL2 AND NOT SDL2_FOUND)
    message(FATAL_ERROR "Failed to enabled required SDL2 backend")
endif ()

# Optionally enable the Wave Writer backend
option(ALSOFT_BACKEND_WAVE "Enable Wave Writer backend" ON)
if (ALSOFT_BACKEND_WAVE)
    set(HAVE_WAVE 1)
    set(ALC_OBJS  ${ALC_OBJS} Alc/backends/wave.c)
    set(BACKENDS  "${BACKENDS} WaveFile,")
endif ()

# This is always available
set(BACKENDS  "${BACKENDS} Null")


find_package(Git)
if (GIT_FOUND AND EXISTS "${OpenAL_SOURCE_DIR}/.git")
    # Get the current working branch and its latest abbreviated commit hash
    add_custom_target(build_version
        ${CMAKE_COMMAND} -D GIT_EXECUTABLE=${GIT_EXECUTABLE}
                         -D LIB_VERSION=${LIB_VERSION}
                         -D SRC=${OpenAL_SOURCE_DIR}/version.h.in
                         -D DST=${OpenAL_BINARY_DIR}/version.h
                         -P ${OpenAL_SOURCE_DIR}/version.cmake
        WORKING_DIRECTORY "${OpenAL_SOURCE_DIR}"
        VERBATIM
    )
else ()
    set(GIT_BRANCH "UNKNOWN")
    set(GIT_COMMIT_HASH "unknown")
    configure_file(
        "${OpenAL_SOURCE_DIR}/version.h.in"
        "${OpenAL_BINARY_DIR}/version.h")
endif ()

set(NATIVE_SRC_DIR  "${OpenAL_SOURCE_DIR}/native-tools/")
set(NATIVE_BIN_DIR  "${OpenAL_BINARY_DIR}/native-tools/")
file(MAKE_DIRECTORY "${NATIVE_BIN_DIR}")

set(BIN2H_COMMAND  "${NATIVE_BIN_DIR}bin2h")
set(BSINCGEN_COMMAND  "${NATIVE_BIN_DIR}bsincgen")
add_subdirectory(native-tools)
#add_custom_command(OUTPUT "${BIN2H_COMMAND}" "${BSINCGEN_COMMAND}"
#    COMMAND ${CMAKE_COMMAND} -G "${CMAKE_GENERATOR}" "${NATIVE_SRC_DIR}"
#    COMMAND ${CMAKE_COMMAND} -E remove "${BIN2H_COMMAND}" "${BSINCGEN_COMMAND}"
#    COMMAND ${CMAKE_COMMAND} --build . --config "Release"
#    WORKING_DIRECTORY "${NATIVE_BIN_DIR}"
#    DEPENDS "${NATIVE_SRC_DIR}CMakeLists.txt"
#    IMPLICIT_DEPENDS C "${NATIVE_SRC_DIR}bin2h.c"
#                     C "${NATIVE_SRC_DIR}bsincgen.c"
#    VERBATIM
#)
add_custom_target(native-tools
    DEPENDS "${BIN2H_COMMAND}" "${BSINCGEN_COMMAND}"
    VERBATIM
)

option(ALSOFT_EMBED_HRTF_DATA "Embed the HRTF data files (increases library footprint)" ON)
if (ALSOFT_EMBED_HRTF_DATA)
    MACRO(make_hrtf_header FILENAME VARNAME)
        set(infile  "${OpenAL_SOURCE_DIR}/hrtf/${FILENAME}")
        set(outfile  "${OpenAL_BINARY_DIR}/${FILENAME}.h")

        add_custom_command(OUTPUT "${outfile}"
            COMMAND "${BIN2H_COMMAND}" "${infile}" "${outfile}" ${VARNAME}
            DEPENDS native-tools "${infile}"
            VERBATIM
        )
        set(ALC_OBJS  ${ALC_OBJS} "${outfile}")
    ENDMACRO()

    make_hrtf_header(default-44100.mhr "hrtf_default_44100")
    make_hrtf_header(default-48000.mhr "hrtf_default_48000")
endif ()

add_custom_command(OUTPUT "${OpenAL_BINARY_DIR}/bsinc_inc.h"
    COMMAND "${BSINCGEN_COMMAND}" "${OpenAL_BINARY_DIR}/bsinc_inc.h"
    DEPENDS native-tools "${NATIVE_SRC_DIR}bsincgen.c"
    VERBATIM
)
set(ALC_OBJS  ${ALC_OBJS} "${OpenAL_BINARY_DIR}/bsinc_inc.h")


if (ALSOFT_UTILS AND NOT ALSOFT_NO_CONFIG_UTIL)
    add_subdirectory(utils/alsoft-config)
endif ()
if (ALSOFT_EXAMPLES)
    if (NOT SDL2_FOUND)
        find_package(SDL2)
    endif ()
    if (SDL2_FOUND)
        find_package(SDL_sound)
        find_package(FFmpeg COMPONENTS AVFORMAT AVCODEC AVUTIL SWSCALE SWRESAMPLE)
    endif ()
endif ()

if (WIN32)
    set(LIBNAME "OpenAL32")
else ()
    set(LIBNAME "openal")
endif ()

# Needed for openal.pc.in
set(prefix ${CMAKE_INSTALL_PREFIX})
set(exec_prefix "\${prefix}")
set(libdir "\${exec_prefix}/${CMAKE_INSTALL_LIBDIR}")
set(bindir "\${exec_prefix}/${CMAKE_INSTALL_BINDIR}")
set(includedir "\${prefix}/${CMAKE_INSTALL_INCLUDEDIR}")
set(PACKAGE_VERSION "${LIB_VERSION}")
set(PKG_CONFIG_CFLAGS )
set(PKG_CONFIG_PRIVATE_LIBS )
if (LIBTYPE STREQUAL "STATIC")
    set(PKG_CONFIG_CFLAGS -DAL_LIBTYPE_STATIC)
    foreach (FLAG  ${LINKER_FLAGS} ${EXTRA_LIBS} ${MATH_LIB})
        # If this is already a linker flag, or is a full path+file, add it
        # as-is. Otherwise, it's a name intended to be dressed as -lname.
        if (FLAG MATCHES "^-.*" OR EXISTS "${FLAG}")
            set(PKG_CONFIG_PRIVATE_LIBS "${PKG_CONFIG_PRIVATE_LIBS} ${FLAG}")
        else ()
            set(PKG_CONFIG_PRIVATE_LIBS "${PKG_CONFIG_PRIVATE_LIBS} -l${FLAG}")
        endif ()
    endforeach ()
endif ()

# End configuration
configure_file(
    "${OpenAL_SOURCE_DIR}/config.h.in"
    "${OpenAL_BINARY_DIR}/config.h")
configure_file(
    "${OpenAL_SOURCE_DIR}/openal.pc.in"
    "${OpenAL_BINARY_DIR}/openal.pc"
    @ONLY)


# Add a static library with common functions used by multiple targets
add_library(common STATIC ${COMMON_OBJS})
target_compile_definitions(common PRIVATE ${CPP_DEFS})
target_compile_options(common PRIVATE ${C_FLAGS})


unset(HAS_ROUTER)
set(IMPL_TARGET OpenAL)
set(COMMON_LIB )
set(SUBSYS_FLAG )

# Build main library
if (LIBTYPE STREQUAL "STATIC")
    set(CPP_DEFS ${CPP_DEFS} AL_LIBTYPE_STATIC)
    if (WIN32 AND ALSOFT_NO_UID_DEFS)
        set(CPP_DEFS ${CPP_DEFS} AL_NO_UID_DEFS)
    endif ()
    add_library(OpenAL STATIC ${COMMON_OBJS} ${OPENAL_OBJS} ${ALC_OBJS})
else ()
    # Make sure to compile the common code with PIC, since it'll be linked into
    # shared libs that needs it.
    set_property(TARGET common PROPERTY POSITION_INDEPENDENT_CODE TRUE)
    set(COMMON_LIB common)

    if (WIN32)
        if (MSVC)
            set(SUBSYS_FLAG ${SUBSYS_FLAG} "/SUBSYSTEM:WINDOWS")
        ELSEIF(CMAKE_COMPILER_IS_GNUCC)
            set(SUBSYS_FLAG ${SUBSYS_FLAG} "-mwindows")
        endif ()
    endif ()

    if (WIN32 AND ALSOFT_BUILD_ROUTER)
        add_library(OpenAL SHARED router/router.c router/router.h router/alc.c router/al.c)
        target_compile_definitions(OpenAL
            PRIVATE AL_BUILD_LIBRARY AL_ALEXT_PROTOTYPES ${CPP_DEFS})
        target_compile_options(OpenAL PRIVATE ${C_FLAGS})
        target_link_libraries(OpenAL PRIVATE ${LINKER_FLAGS} ${COMMON_LIB})
        set_target_properties(OpenAL PROPERTIES PREFIX "")
        set_target_properties(OpenAL PROPERTIES OUTPUT_NAME ${LIBNAME})
        if (TARGET build_version)
            add_dependencies(OpenAL build_version)
        endif ()
        set(HAS_ROUTER 1)

        set(LIBNAME "soft_oal")
        set(IMPL_TARGET soft_oal)
    endif ()

    add_library(${IMPL_TARGET} SHARED ${OPENAL_OBJS} ${ALC_OBJS})
    if (WIN32)
        set_target_properties(${IMPL_TARGET} PROPERTIES PREFIX "")
    endif ()
endif ()

set_target_properties(${IMPL_TARGET} PROPERTIES OUTPUT_NAME ${LIBNAME}
    VERSION ${LIB_VERSION}
    SOVERSION ${LIB_MAJOR_VERSION}
)

target_compile_definitions(${IMPL_TARGET} PRIVATE AL_BUILD_LIBRARY AL_ALEXT_PROTOTYPES ${CPP_DEFS})
target_include_directories(${IMPL_TARGET} PRIVATE "${OpenAL_SOURCE_DIR}/OpenAL32/Include" "${OpenAL_SOURCE_DIR}/Alc" ${INC_PATHS})
target_compile_options(${IMPL_TARGET} PRIVATE ${C_FLAGS})
target_link_libraries(${IMPL_TARGET} PRIVATE ${LINKER_FLAGS} ${COMMON_LIB} ${EXTRA_LIBS} ${MATH_LIB})

if (TARGET build_version)
    add_dependencies(${IMPL_TARGET} build_version)
endif ()

if (WIN32 AND MINGW AND ALSOFT_BUILD_IMPORT_LIB AND NOT LIBTYPE STREQUAL "STATIC")
    find_program(SED_EXECUTABLE NAMES sed DOC "sed executable")
    find_program(DLLTOOL_EXECUTABLE NAMES "${DLLTOOL}" DOC "dlltool executable")
    if (NOT SED_EXECUTABLE OR NOT DLLTOOL_EXECUTABLE)
        message(STATUS "")
        if (NOT SED_EXECUTABLE)
            message(STATUS "WARNING: Cannot find sed, disabling .def/.lib generation")
        endif ()
        if (NOT DLLTOOL_EXECUTABLE)
            message(STATUS "WARNING: Cannot find dlltool, disabling .def/.lib generation")
        endif ()
    else ()
        set_property(TARGET OpenAL APPEND_STRING PROPERTY LINK_FLAGS
                        " -Wl,--output-def,OpenAL32.def")
        add_custom_command(TARGET OpenAL POST_BUILD
            COMMAND "${SED_EXECUTABLE}" -i -e "s/ @[^ ]*//" OpenAL32.def
            COMMAND "${DLLTOOL_EXECUTABLE}" -d OpenAL32.def -l OpenAL32.lib -D OpenAL32.dll
            COMMENT "Stripping ordinals from OpenAL32.def and generating OpenAL32.lib..."
            VERBATIM
        )
    endif ()
endif ()

if (ALSOFT_INSTALL)
    install(TARGETS OpenAL
            EXPORT OpenAL-targets
            RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
            LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
            ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
            INCLUDES DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/OpenAL" "${CMAKE_INSTALL_INCLUDEDIR}/OpenAL/AL"
    )
    
    EXPORT(TARGETS OpenAL
           NAMESPACE OpenAL::
           FILE OpenALConfig.cmake)
           
    install(EXPORT OpenAL-targets
            NAMESPACE OpenAL::
            DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/OpenAL")
            
    install(FILES include/AL/al.h
                  include/AL/alc.h
                  include/AL/alext.h
                  include/AL/efx.h
                  include/AL/efx-creative.h
                  include/AL/efx-presets.h
            DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/OpenAL/AL"
    )
    
    install(FILES "${OpenAL_BINARY_DIR}/openal.pc"
            DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/OpenAL")
            
    if (TARGET soft_oal)
        install(TARGETS soft_oal
                RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
        )
    endif ()
endif ()


if (HAS_ROUTER)
    message(STATUS "")
    message(STATUS "Building DLL router")
endif ()

message(STATUS "")
message(STATUS "Building OpenAL with support for the following backends:")
message(STATUS "    ${BACKENDS}")
message(STATUS "")
message(STATUS "Building with support for CPU extensions:")
message(STATUS "    ${CPU_EXTS}")
message(STATUS "")
if (FPMATH_SET)
    message(STATUS "Building with SSE${FPMATH_SET} codegen")
    message(STATUS "")
endif ()

if (WIN32)
    if (NOT HAVE_DSOUND)
        message(STATUS "WARNING: Building the Windows version without DirectSound output")
        message(STATUS "         This is probably NOT what you want!")
        message(STATUS "")
    endif ()
endif ()

if (ALSOFT_EMBED_HRTF_DATA)
    message(STATUS "Embedding HRTF datasets")
    message(STATUS "")
endif ()

# Install alsoft.conf configuration file
if (ALSOFT_CONFIG)
    install(FILES alsoftrc.sample
            DESTINATION ${CMAKE_INSTALL_DATADIR}/OpenAL
    )
    message(STATUS "Installing sample configuration")
    message(STATUS "")
endif ()

# Install HRTF definitions
if (ALSOFT_HRTF_DEFS)
    install(FILES hrtf/default-44100.mhr
                  hrtf/default-48000.mhr
            DESTINATION ${CMAKE_INSTALL_DATADIR}/OpenAL/hrtf
    )
    message(STATUS "Installing HRTF definitions")
    message(STATUS "")
endif ()

# Install AmbDec presets
if (ALSOFT_AMBDEC_PRESETS)
    install(FILES presets/3D7.1.ambdec
                  presets/hexagon.ambdec
                  presets/itu5.1.ambdec
                  presets/itu5.1-nocenter.ambdec
                  presets/rectangle.ambdec
                  presets/square.ambdec
                  presets/presets.txt
            DESTINATION ${CMAKE_INSTALL_DATADIR}/OpenAL/presets
    )
    message(STATUS "Installing AmbDec presets")
    message(STATUS "")
endif ()

if (ALSOFT_UTILS)
    add_executable(openal-info utils/openal-info.c)
    target_compile_options(openal-info PRIVATE ${C_FLAGS})
    target_link_libraries(openal-info PRIVATE ${LINKER_FLAGS} OpenAL)

    set(MAKEHRTF_SRCS  utils/makehrtf.c)
    if (NOT HAVE_GETOPT)
        set(MAKEHRTF_SRCS  ${MAKEHRTF_SRCS} utils/getopt.c utils/getopt.h)
    endif ()
    add_executable(makehrtf ${MAKEHRTF_SRCS})
    target_compile_definitions(makehrtf PRIVATE ${CPP_DEFS})
    target_compile_options(makehrtf PRIVATE ${C_FLAGS})
    if (HAVE_LIBM)
        target_link_libraries(makehrtf PRIVATE ${LINKER_FLAGS} m)
    endif ()

    if (ALSOFT_INSTALL)
        install(TARGETS openal-info makehrtf
                RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
                LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
                ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        )
    endif ()

    message(STATUS "Building utility programs")
    if (TARGET alsoft-config)
        message(STATUS "Building configuration program")
    endif ()
    message(STATUS "")
endif ()

if (ALSOFT_TESTS)
    set(TEST_COMMON_OBJS  examples/common/alhelpers.c)

    add_executable(altonegen examples/altonegen.c ${TEST_COMMON_OBJS})
    target_compile_definitions(altonegen PRIVATE ${CPP_DEFS})
    target_compile_options(altonegen PRIVATE ${C_FLAGS})
    target_link_libraries(altonegen PRIVATE ${LINKER_FLAGS} common OpenAL ${MATH_LIB})

    if (ALSOFT_INSTALL)
        install(TARGETS altonegen
                RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
                LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
                ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        )
    endif ()

    message(STATUS "Building test programs")
    message(STATUS "")
endif ()

if (ALSOFT_EXAMPLES)
    add_executable(alrecord examples/alrecord.c)
    target_compile_definitions(alrecord PRIVATE ${CPP_DEFS})
    target_compile_options(alrecord PRIVATE ${C_FLAGS})
    target_link_libraries(alrecord PRIVATE ${LINKER_FLAGS} common OpenAL)

    if (ALSOFT_INSTALL)
        install(TARGETS alrecord
                RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
                LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
                ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        )
    endif ()

    message(STATUS "Building example programs")

    if (SDL2_FOUND)
        if (SDL_SOUND_FOUND)
            # Add a static library with common functions used by multiple targets
            add_library(ex-common STATIC examples/common/alhelpers.c)
            target_compile_definitions(ex-common PRIVATE ${CPP_DEFS})
            target_compile_options(ex-common PRIVATE ${C_FLAGS})

            add_executable(alplay examples/alplay.c)
            target_compile_definitions(alplay PRIVATE ${CPP_DEFS})
            target_include_directories(alplay
                PRIVATE ${SDL2_INCLUDE_DIR} ${SDL_SOUND_INCLUDE_DIR})
            target_compile_options(alplay PRIVATE ${C_FLAGS})
            target_link_libraries(alplay
                PRIVATE ${LINKER_FLAGS} ${SDL_SOUND_LIBRARIES} ${SDL2_LIBRARY} ex-common common
                        OpenAL)

            add_executable(alstream examples/alstream.c)
            target_compile_definitions(alstream PRIVATE ${CPP_DEFS})
            target_include_directories(alstream
                PRIVATE ${SDL2_INCLUDE_DIR} ${SDL_SOUND_INCLUDE_DIR})
            target_compile_options(alstream PRIVATE ${C_FLAGS})
            target_link_libraries(alstream
                PRIVATE ${LINKER_FLAGS} ${SDL_SOUND_LIBRARIES} ${SDL2_LIBRARY} ex-common common
                        OpenAL)

            add_executable(alreverb examples/alreverb.c)
            target_compile_definitions(alreverb PRIVATE ${CPP_DEFS})
            target_include_directories(alreverb
                PRIVATE ${SDL2_INCLUDE_DIR} ${SDL_SOUND_INCLUDE_DIR})
            target_compile_options(alreverb PRIVATE ${C_FLAGS})
            target_link_libraries(alreverb
                PRIVATE ${LINKER_FLAGS} ${SDL_SOUND_LIBRARIES} ${SDL2_LIBRARY} ex-common common
                        OpenAL)

            add_executable(almultireverb examples/almultireverb.c)
            target_compile_definitions(almultireverb PRIVATE ${CPP_DEFS})
            target_include_directories(almultireverb
                PRIVATE ${SDL2_INCLUDE_DIR} ${SDL_SOUND_INCLUDE_DIR})
            target_compile_options(almultireverb PRIVATE ${C_FLAGS})
            target_link_libraries(almultireverb
                PRIVATE ${LINKER_FLAGS} ${SDL_SOUND_LIBRARIES} ${SDL2_LIBRARY} ex-common common
                        OpenAL ${MATH_LIB})

            add_executable(allatency examples/allatency.c)
            target_compile_definitions(allatency PRIVATE ${CPP_DEFS})
            target_include_directories(allatency
                PRIVATE ${SDL2_INCLUDE_DIR} ${SDL_SOUND_INCLUDE_DIR})
            target_compile_options(allatency PRIVATE ${C_FLAGS})
            target_link_libraries(allatency
                PRIVATE ${LINKER_FLAGS} ${SDL_SOUND_LIBRARIES} ${SDL2_LIBRARY} ex-common common
                        OpenAL)

            add_executable(alloopback examples/alloopback.c)
            target_compile_definitions(alloopback PRIVATE ${CPP_DEFS})
            target_include_directories(alloopback
                PRIVATE ${SDL2_INCLUDE_DIR} ${SDL_SOUND_INCLUDE_DIR})
            target_compile_options(alloopback PRIVATE ${C_FLAGS})
            target_link_libraries(alloopback
                PRIVATE ${LINKER_FLAGS} ${SDL_SOUND_LIBRARIES} ${SDL2_LIBRARY} ex-common common
                        OpenAL ${MATH_LIB})

            add_executable(alhrtf examples/alhrtf.c)
            target_compile_definitions(alhrtf PRIVATE ${CPP_DEFS})
            target_include_directories(alhrtf
                PRIVATE ${SDL2_INCLUDE_DIR} ${SDL_SOUND_INCLUDE_DIR})
            target_compile_options(alhrtf PRIVATE ${C_FLAGS})
            target_link_libraries(alhrtf
                PRIVATE ${LINKER_FLAGS} ${SDL_SOUND_LIBRARIES} ${SDL2_LIBRARY} ex-common common
                        OpenAL ${MATH_LIB})

            if (ALSOFT_INSTALL)
                install(TARGETS alplay alstream alreverb almultireverb allatency alloopback alhrtf
                        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
                        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
                        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
                )
            endif ()

            message(STATUS "Building SDL_sound example programs")
        endif ()

        set(FFVER_OK FALSE)
        if (FFMPEG_FOUND)
            set(FFVER_OK TRUE)
            if (AVFORMAT_VERSION VERSION_LESS "57.56.101")
                message(STATUS "libavformat is too old! (${AVFORMAT_VERSION}, wanted 57.56.101)")
                set(FFVER_OK FALSE)
            endif ()
            if (AVCODEC_VERSION VERSION_LESS "57.64.101")
                message(STATUS "libavcodec is too old! (${AVCODEC_VERSION}, wanted 57.64.101)")
                set(FFVER_OK FALSE)
            endif ()
            if (AVUTIL_VERSION VERSION_LESS "55.34.101")
                message(STATUS "libavutil is too old! (${AVUTIL_VERSION}, wanted 55.34.101)")
                set(FFVER_OK FALSE)
            endif ()
            if (SWSCALE_VERSION VERSION_LESS "4.2.100")
                message(STATUS "libswscale is too old! (${SWSCALE_VERSION}, wanted 4.2.100)")
                set(FFVER_OK FALSE)
            endif ()
            if (SWRESAMPLE_VERSION VERSION_LESS "2.3.100")
                message(STATUS "libswresample is too old! (${SWRESAMPLE_VERSION}, wanted 2.3.100)")
                set(FFVER_OK FALSE)
            endif ()
        endif ()
        if (FFVER_OK)
            add_executable(alffplay examples/alffplay.cpp)
            target_compile_definitions(alffplay PRIVATE ${CPP_DEFS})
            target_include_directories(alffplay
                PRIVATE ${SDL2_INCLUDE_DIR} ${FFMPEG_INCLUDE_DIRS})
            target_compile_options(alffplay PRIVATE ${C_FLAGS})
            target_link_libraries(alffplay
                PRIVATE ${LINKER_FLAGS} ${SDL2_LIBRARY} ${FFMPEG_LIBRARIES} ex-common common
                        OpenAL)

            if (ALSOFT_INSTALL)
                install(TARGETS alffplay
                        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
                        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
                        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
                )
            endif ()
            message(STATUS "Building SDL+FFmpeg example programs")
        endif ()
        message(STATUS "")
    endif ()
endif ()
