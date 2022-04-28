#***************************************************************************
# Copyright (C) 2017-2020 Nathan Moinvaziri
#                         https://github.com/zlib-ng/minizip-ng
# Copyright (C)      2016 Matthias Schmieder
#                         schmieder.matthias@gmail.com
#***************************************************************************

# edited 20220429, axki

cmake_minimum_required(VERSION 3.13)

add_compile_options($<$<C_COMPILER_ID:MSVC>:/MP>)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/cmake")

message(STATUS "Using CMake version ${CMAKE_VERSION}")

set(ZSTD_BUILD_SHARED OFF)
set(ZSTD_BUILD_PROGRAMS OFF)
set(ZSTD_BUILD_CONTRIB OFF)
set(TUKLIB_FAST_UNALIGNED_ACCESS ON)
set(TUKLIB_USE_UNSAFE_TYPE_PUNNING OFF)
set(ZSTD_USE_STATIC_RUNTIME ON)
set(ZSTD_PROGRAMS_LINK_SHARED OFF)

set(ACTIVATE_MULTITHREADED_COMPILATION ON)
set(BUILD_SHARED_LIBS OFF)  # build as static lib
# Compatibility options
set(MZ_COMPAT ON)
# Compression library options
set(MZ_ZLIB ON)
set(MZ_BZIP2 ON)
set(MZ_LZMA ON)
set(MZ_ZSTD ON)
set(MZ_LIBCOMP OFF)
set(MZ_FETCH_LIBS ON)
set(MZ_FORCE_FETCH_LIBS ON)
# Encryption support options
set(MZ_PKCRYPT ON)
set(MZ_WZAES ON)
set(MZ_OPENSSL OFF)
set(MZ_LIBBSD OFF)
set(MZ_SIGNING ON)
# Character conversion options
set(MZ_ICONV ON)
# Code generation options
set(MZ_COMPRESS_ONLY OFF)
set(MZ_DECOMPRESS_ONLY OFF)
set(MZ_FILE32_API OFF)
# Build and continuous integration options
set(MZ_BUILD_TESTS OFF CACHE BOOL "" FORCE)
set(MZ_BUILD_UNIT_TESTS OFF)
set(MZ_BUILD_FUZZ_TESTS OFF)
set(MZ_CODE_COVERAGE OFF)

# Package management options
if(NOT MZ_COMPAT AND NOT DEFINED MZ_PROJECT_SUFFIX)
    set(MZ_PROJECT_SUFFIX "-ng" CACHE STRING "Project name suffix for package managers")
else()
    set(MZ_PROJECT_SUFFIX "" CACHE STRING "Project name suffix for package managers")
endif()

mark_as_advanced(MZ_FILE32_API MZ_PROJECT_SUFFIX)


if(POLICY CMP0054)
    cmake_policy(SET CMP0054 NEW)
endif()

# ZLIB_ROOT - Parent directory of zlib installation
# BZIP2_ROOT - Parent directory of BZip2 installation
# OPENSSL_ROOT - Parent directory of OpenSSL installation

enable_language(C)

# Library version
set(VERSION "3.0.6")

# API version
set(SOVERSION "3")

include(CheckLibraryExists)
include(CheckSymbolExists)
include(CheckFunctionExists)
include(CheckIncludeFile)
include(CheckTypeSize)
include(GNUInstallDirs)
include(FeatureSummary)

include(cmake/clone-repo.cmake)

set(INSTALL_LIB_DIR ${CMAKE_INSTALL_LIBDIR} CACHE PATH "Installation directory for libraries")
set(INSTALL_INC_DIR ${CMAKE_INSTALL_INCLUDEDIR} CACHE PATH "Installation directory for headers")

set(STDLIB_DEF)
set(MINIZIP_DEF)
set(MINIZIP_INC)
set(MINIZIP_LIB)
set(MINIZIP_LBD)
set(MINIZIP_DEP)
set(MINIZIP_DEP_PKG)
set(MINIZIP_LFG)

# Initial source files
set(MINIZIP_SRC
    mz_crypt.c
    mz_os.c
    mz_strm.c
    mz_strm_buf.c
    mz_strm_mem.c
    mz_strm_split.c
    mz_zip.c
    mz_zip_rw.c)

# Initial header files
set(MINIZIP_HDR
    mz.h
    mz_os.h
    mz_crypt.h
    mz_strm.h
    mz_strm_buf.h
    mz_strm_mem.h
    mz_strm_split.h
    mz_strm_os.h
    mz_zip.h
    mz_zip_rw.h)

set(PC_PRIVATE_LIBS)

# Check for system includes
check_include_file(stdint.h   HAVE_STDINT_H)
check_include_file(inttypes.h HAVE_INTTYPES_H)

if(HAVE_STDINT_H)
    list(APPEND STDLIB_DEF -DHAVE_STDINT_H)
endif()
if(HAVE_INTTYPES_H)
    list(APPEND STDLIB_DEF -DHAVE_INTTYPES_H)
endif()

# Check for large file support
check_type_size(off64_t OFF64_T)
if(HAVE_OFF64_T)
    list(APPEND STDLIB_DEF -D__USE_LARGEFILE64)
    list(APPEND STDLIB_DEF -D_LARGEFILE64_SOURCE)
endif()
# Check for fseeko support
check_function_exists(fseeko HAVE_FSEEKO)
if(NOT HAVE_FSEEKO)
    list(APPEND STDLIB_DEF -DNO_FSEEKO)
endif()

set(MZ_LIBCOMP OFF)

if(MZ_ZLIB)
    # Check if zlib is present
    if(NOT MZ_FORCE_FETCH_LIBS)
        find_package(ZLIBNG QUIET)
        find_package(ZLIB QUIET)
        set(ZLIB_VERSION ${ZLIB_VERSION_STRING})
    endif()

    if(ZLIBNG_FOUND AND NOT MZ_FORCE_FETCH_LIBS)
        message(STATUS "Using ZLIBNG")

        list(APPEND MINIZIP_INC ${ZLIBNG_INCLUDE_DIRS})
        list(APPEND MINIZIP_LIB ${ZLIBNG_LIBRARIES})
        list(APPEND MINIZIP_LBD ${ZLIBNG_LIBRARY_DIRS})

        set(PC_PRIVATE_LIBS " -lz-ng")
        set(ZLIB_COMPAT OFF)
    elseif(ZLIB_FOUND AND NOT MZ_FORCE_FETCH_LIBS)
        message(STATUS "Using ZLIB ${ZLIB_VERSION}")

        list(APPEND MINIZIP_INC ${ZLIB_INCLUDE_DIRS})
        list(APPEND MINIZIP_LIB ${ZLIB_LIBRARIES})
        list(APPEND MINIZIP_LBD ${ZLIB_LIBRARY_DIRS})

        set(PC_PRIVATE_LIBS " -lz")
        set(ZLIB_COMPAT ON)
    elseif(MZ_FETCH_LIBS)
        clone_repo(zlib https://github.com/madler/zlib)

        # Don't automatically add all targets to the solution
        add_subdirectory(${ZLIB_SOURCE_DIR} ${ZLIB_BINARY_DIR} EXCLUDE_FROM_ALL)

        list(APPEND MINIZIP_INC ${ZLIB_SOURCE_DIR})
        list(APPEND MINIZIP_INC ${ZLIB_BINARY_DIR})

        # Have to add zlib to install targets
        list(APPEND MINIZIP_DEP zlibstatic)

        if(EXISTS "${ZLIB_BINARY_DIR}/zlib-ng.h")
            message(STATUS "ZLIB repository detected as ZLIBNG")
            set(ZLIB_COMPAT OFF)
        else()
            set(ZLIB_COMPAT ON)
        endif()
    else()
        message(STATUS "ZLIB library not found")

        set(MZ_ZLIB OFF)
    endif()

    if(MZ_ZLIB)
        list(APPEND MINIZIP_DEF -DHAVE_ZLIB)
        if(ZLIB_COMPAT)
            list(APPEND MINIZIP_DEF -DZLIB_COMPAT)
        endif()
        if(ZLIBNG_FOUND OR NOT ZLIB_COMPAT)
            list(APPEND MINIZIP_DEP_PKG ZLIBNG)
        elseif(ZLIB_FOUND)
            list(APPEND MINIZIP_DEP_PKG ZLIB)
        endif()
        list(APPEND MINIZIP_SRC mz_strm_zlib.c)
        list(APPEND MINIZIP_HDR mz_strm_zlib.h)
    endif()
endif()

if(MZ_BZIP2)
    # Check if bzip2 is present
    if(NOT MZ_FORCE_FETCH_LIBS)
        find_package(BZip2 QUIET)
    endif()

    if(BZIP2_FOUND AND NOT MZ_FORCE_FETCH_LIBS)
        message(STATUS "Using BZIP2 ${BZIP2_VERSION_STRING}")

        list(APPEND MINIZIP_INC ${BZIP2_INCLUDE_DIRS})
        list(APPEND MINIZIP_LIB ${BZIP2_LIBRARIES})
        list(APPEND MINIZIP_LBD ${BZIP2_LIBRARY_DIRS})

        set(PC_PRIVATE_LIBS "${PC_PRIVATE_LIBS} -lbzip2")
    elseif(MZ_FETCH_LIBS)
        clone_repo(bzip2 https://sourceware.org/git/bzip2.git)

        # BZip2 repository does not support cmake so we have to create
        # the bzip2 library ourselves
        set(BZIP2_SRC
            ${BZIP2_SOURCE_DIR}/blocksort.c
            ${BZIP2_SOURCE_DIR}/bzlib.c
            ${BZIP2_SOURCE_DIR}/compress.c
            ${BZIP2_SOURCE_DIR}/crctable.c
            ${BZIP2_SOURCE_DIR}/decompress.c
            ${BZIP2_SOURCE_DIR}/huffman.c
            ${BZIP2_SOURCE_DIR}/randtable.c)

        set(BZIP2_HDR
            ${BZIP2_SOURCE_DIR}/bzlib.h
            ${BZIP2_SOURCE_DIR}/bzlib_private.h)

        add_library(bzip2 STATIC ${BZIP2_SRC} ${BZIP2_HDR})

        target_compile_definitions(bzip2 PRIVATE -DBZ_NO_STDIO)

        list(APPEND MINIZIP_DEP bzip2)
        list(APPEND MINIZIP_INC ${BZIP2_SOURCE_DIR})
    else()
        message(STATUS "BZip2 library not found")

        set(MZ_BZIP2 OFF)
    endif()

    if(MZ_BZIP2)
        list(APPEND MINIZIP_DEP_PKG BZip2)
        list(APPEND MINIZIP_DEF -DHAVE_BZIP2)
        list(APPEND MINIZIP_SRC mz_strm_bzip.c)
        list(APPEND MINIZIP_HDR mz_strm_bzip.h)
    endif()
endif()

if(MZ_LZMA)
    # Check if liblzma is present
    if(NOT MZ_FORCE_FETCH_LIBS)
        find_package(PkgConfig QUIET)
        if(PKGCONFIG_FOUND)
            pkg_check_modules(LIBLZMA liblzma)
        endif()
        if(NOT LIBLZMA_FOUND)
            find_package(LibLZMA QUIET)
            set(LIBLZMA_VERSION ${LIBLZMA_VERSION_STRING})
        endif()
    endif()

    if(LIBLZMA_FOUND AND NOT MZ_FORCE_FETCH_LIBS)
        message(STATUS "Using LZMA ${LIBLZMA_VERSION}")

        list(APPEND MINIZIP_INC ${LIBLZMA_INCLUDE_DIRS})
        list(APPEND MINIZIP_LIB ${LIBLZMA_LIBRARIES})

        set(PC_PRIVATE_LIBS "${PC_PRIVATE_LIBS} -lliblzma")
    elseif(MZ_FETCH_LIBS)
        clone_repo(liblzma https://git.tukaani.org/xz.git)

        # Don't automatically add all targets to the solution
        add_subdirectory(${LIBLZMA_SOURCE_DIR} ${LIBLZMA_BINARY_DIR} EXCLUDE_FROM_ALL)

        list(APPEND MINIZIP_INC ${LIBLZMA_SOURCE_DIR}/src/liblzma/api)
        list(APPEND MINIZIP_DEP liblzma)
        list(APPEND MINIZIP_LIB ${LIBLZMA_TARGET})
    else()
        message(STATUS "LibLZMA library not found")

        set(MZ_LZMA OFF)
    endif()

    if(MZ_LZMA)
        list(APPEND MINIZIP_DEP_PKG LibLZMA)
        list(APPEND MINIZIP_DEF -DHAVE_LZMA -DLZMA_API_STATIC)
        list(APPEND MINIZIP_SRC mz_strm_lzma.c)
        list(APPEND MINIZIP_HDR mz_strm_lzma.h)
    endif()
endif()

if(MZ_ZSTD)
    # Check if zstd is present
    if(NOT MZ_FORCE_FETCH_LIBS)
        find_package(PkgConfig QUIET)
        if(PKGCONFIG_FOUND)
            pkg_check_modules(ZSTD libzstd)
        endif()
        if(NOT ZSTD_FOUND)
            find_package(ZSTD QUIET)
            if(ZSTD_FOUND)
                if(TARGET zstd::libzstd_static)
                    list(APPEND ZSTD_LIBRARIES zstd::libzstd_static)
                else()
                    list(APPEND ZSTD_LIBRARIES zstd::libzstd_shared)
                endif()
            endif()
        endif()
    endif()

    if(ZSTD_FOUND AND NOT MZ_FORCE_FETCH_LIBS)
        message(STATUS "Using ZSTD ${ZSTD_VERSION}")

        list(APPEND MINIZIP_INC ${ZSTD_INCLUDE_DIRS})
        list(APPEND MINIZIP_LIB ${ZSTD_LIBRARIES})
        list(APPEND MINIZIP_LBD ${ZSTD_LIBRARY_DIRS})

        set(PC_PRIVATE_LIBS "${PC_PRIVATE_LIBS} -lzstd")
    elseif(MZ_FETCH_LIBS)
        clone_repo(zstd https://github.com/facebook/zstd)

        # Don't automatically add all targets to the solution
        add_subdirectory(${ZSTD_SOURCE_DIR}/build/cmake ${ZSTD_BINARY_DIR} EXCLUDE_FROM_ALL)

        list(APPEND MINIZIP_INC ${ZSTD_SOURCE_DIR}/lib)

        list(APPEND MINIZIP_DEP libzstd_static)

    else()
        message(STATUS "ZSTD library not found")

        set(MZ_ZSTD OFF)
    endif()

    if(MZ_ZSTD)
        list(APPEND MINIZIP_DEP_PKG zstd)
        list(APPEND MINIZIP_DEF -DHAVE_ZSTD)
        list(APPEND MINIZIP_SRC mz_strm_zstd.c)
        list(APPEND MINIZIP_HDR mz_strm_zstd.h)
    endif()
endif()

if(NOT MZ_LIBCOMP AND NOT MZ_ZLIB AND NOT MZ_ZSTD AND NOT MZ_BZIP2 AND NOT MZ_LZMA)
    message(STATUS "Compression not supported due to missing libraries")

    list(APPEND MINIZIP_DEF -DMZ_ZIP_NO_DECOMPRESSION)
    list(APPEND MINIZIP_DEF -DMZ_ZIP_NO_COMPRESSION)
endif()

if(MZ_OPENSSL)
    # Check to see if openssl installation is present
    find_package(PkgConfig)
    if(PKGCONFIG_FOUND)
        pkg_check_modules(OPENSSL openssl)
    endif()
    if(NOT OPENSSL_FOUND)
        find_package(OpenSSL)
    endif()

    if(OPENSSL_FOUND)
        message(STATUS "Using OpenSSL ${OPENSSL_VERSION}")

        list(APPEND MINIZIP_SRC mz_crypt_openssl.c)
        list(APPEND MINIZIP_LIB ${OPENSSL_LIBRARIES})
        list(APPEND MINIZIP_LBD ${OPENSSL_LIBRARY_DIRS})

        if(OPENSSL_INCLUDE_DIRS)
            list(APPEND MINIZIP_INC ${OPENSSL_INCLUDE_DIRS})
        endif()
        if(OPENSSL_INCLUDE_DIR)
            list(APPEND MINIZIP_INC ${OPENSSL_INCLUDE_DIR})
        endif()
    else()
        message(STATUS "OpenSSL library not found")

        set(MZ_OPENSSL OFF)
    endif()
endif()

# Windows specific
if(WIN32)
    list(APPEND MINIZIP_DEF -D_CRT_SECURE_NO_DEPRECATE)
    list(APPEND MINIZIP_SRC mz_os_win32.c mz_strm_os_win32.c)

    if(MZ_PKCRYPT OR MZ_WZAES OR MZ_SIGNING)
        if(MZ_OPENSSL)
            list(APPEND MINIZIP_DEP_PKG OpenSSL)
        else()
            message(STATUS "Using CryptoAPI")

            list(APPEND MINIZIP_SRC mz_crypt_win32.c)
            list(APPEND MINIZIP_LIB crypt32.lib)
        endif()
    else()
        list(APPEND MINIZIP_DEF -DMZ_ZIP_NO_CRYPTO)
    endif()
endif()
if(CMAKE_SYSTEM_NAME STREQUAL "WindowsStore")
    list(APPEND MINIZIP_DEF -DMZ_WINRT_API)
endif()

set(MZ_LIBBSD OFF)
set(MZ_ICONV OFF)

# Setup predefined macros
if(NOT MZ_PKCRYPT AND NOT MZ_WZAES)
    list(APPEND MINIZIP_DEF -DMZ_ZIP_NO_ENCRYPTION)
endif()
if(MZ_SIGNING)
    list(APPEND MINIZIP_DEF -DMZ_ZIP_SIGNING)
endif()
if(MZ_FILE32_API)
    list(APPEND MINIZIP_DEF -DMZ_FILE32_API)
endif()

# Include traditional PKWare encryption
if(MZ_PKCRYPT)
    list(APPEND MINIZIP_DEF -DHAVE_PKCRYPT)
    list(APPEND MINIZIP_SRC mz_strm_pkcrypt.c)
    list(APPEND MINIZIP_HDR mz_strm_pkcrypt.h)
endif()

# Include WinZIP AES encryption
if(MZ_WZAES)
    list(APPEND MINIZIP_DEF -DHAVE_WZAES)
    list(APPEND MINIZIP_SRC mz_strm_wzaes.c)
    list(APPEND MINIZIP_HDR mz_strm_wzaes.h)
endif()

# Include compatibility layer
if(MZ_COMPAT)
    set(FILE_H "zip.h")
    set(MZ_COMPAT_FILE "MZ_COMPAT_ZIP")
    configure_file(mz_compat_shim.h.in zip.h)

    set(FILE_H "unzip.h")
    set(MZ_COMPAT_FILE "MZ_COMPAT_UNZIP")
    configure_file(mz_compat_shim.h.in unzip.h)

    if(MZ_COMPAT_VERSION)
        list(APPEND MINIZIP_DEF -DMZ_COMPAT_VERSION=${MZ_COMPAT_VERSION})
    endif()
    list(APPEND MINIZIP_SRC mz_compat.c)
    list(APPEND MINIZIP_HDR mz_compat.h ${CMAKE_CURRENT_BINARY_DIR}/zip.h ${CMAKE_CURRENT_BINARY_DIR}/unzip.h)
endif()

# Set compiler options
if(MSVC)
    add_compile_options(
        $<$<CONFIG:Debug>:/Zi>
        $<$<CONFIG:Debug>:/Od>
        $<$<CONFIG:Debug>:/W3>
        $<$<CONFIG:Release>:/Ox>
        $<$<CONFIG:Release>:/Os>)
else()
    add_compile_options(
        $<$<CONFIG:Debug>:-g>
        $<$<CONFIG:Debug>:-Wall>
        $<$<CONFIG:Release>:-Os>)
endif()

list(APPEND MINIZIP_INC ${CMAKE_CURRENT_SOURCE_DIR})

# Create minizip library
project(minizip${MZ_PROJECT_SUFFIX} LANGUAGES C VERSION ${VERSION})

if(NOT ${MZ_PROJECT_SUFFIX} STREQUAL "")
    message(STATUS "Project configured as ${PROJECT_NAME}")
endif()

set(MINIZIP_PC ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}.pc)
configure_file(minizip.pc.cmakein ${MINIZIP_PC} @ONLY)

set(INSTALL_CMAKE_DIR "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}"
    CACHE PATH "Installation directory for cmake files.")
set(INSTALL_PKGCONFIG_DIR "${CMAKE_INSTALL_LIBDIR}/pkgconfig"
    CACHE PATH "Installation directory for pkgconfig (.pc) files")

add_library(${PROJECT_NAME} ${MINIZIP_SRC} ${MINIZIP_HDR})

set_target_properties(${PROJECT_NAME} PROPERTIES
                      VERSION ${VERSION}
                      SOVERSION ${SOVERSION}
                      LINKER_LANGUAGE C
                      DEFINE_SYMBOL "MZ_EXPORTS")

if(MINIZIP_LFG)
    set_target_properties(${PROJECT_NAME} PROPERTIES LINK_FLAGS ${MINIZIP_LFG})
endif()
if(MSVC)
    # VS debugger has problems when executable and static library are named the same
    set_target_properties(${PROJECT_NAME} PROPERTIES OUTPUT_NAME lib${PROJECT_NAME})
endif()
if(NOT RISCOS AND NOT PSP)
    set_target_properties(${PROJECT_NAME} PROPERTIES POSITION_INDEPENDENT_CODE 1)
endif()
if(MZ_LZMA)
    set_target_properties(${PROJECT_NAME} PROPERTIES C_STANDARD 99)
endif()

target_link_libraries(${PROJECT_NAME} PUBLIC ${MINIZIP_LIB} ${MINIZIP_DEP})
target_link_directories(${PROJECT_NAME} PUBLIC ${MINIZIP_LBD})
target_compile_definitions(${PROJECT_NAME} PRIVATE ${STDLIB_DEF} ${MINIZIP_DEF})
target_include_directories(${PROJECT_NAME} PRIVATE ${MINIZIP_INC})
target_include_directories(${PROJECT_NAME} PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>)
if(MZ_COMPAT)
    target_include_directories(${PROJECT_NAME} PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>)
endif()

# Create minizip alias
add_library(MINIZIP::minizip ALIAS ${PROJECT_NAME})

# Install files
if(NOT SKIP_INSTALL_LIBRARIES AND NOT SKIP_INSTALL_ALL)
    target_include_directories(${PROJECT_NAME} PUBLIC
        $<INSTALL_INTERFACE:${INSTALL_INC_DIR}>)

    INSTALL(TARGETS ${PROJECT_NAME} ${MINIZIP_DEP}
    CONFIGURATIONS Debug
    INCLUDES DESTINATION "${INSTALL_INC_DIR}/Debug"
    LIBRARY DESTINATION "${INSTALL_LIB_DIR}/Debug"
    ARCHIVE DESTINATION "${INSTALL_LIB_DIR}/Debug")

    INSTALL(TARGETS ${PROJECT_NAME} ${MINIZIP_DEP}
    EXPORT ${PROJECT_NAME}
    CONFIGURATIONS RelWithDebInfo
    INCLUDES DESTINATION "${INSTALL_INC_DIR}/Release"
    LIBRARY DESTINATION "${INSTALL_LIB_DIR}/Release"
    ARCHIVE DESTINATION "${INSTALL_LIB_DIR}/Release")
endif()
if(NOT SKIP_INSTALL_HDR AND NOT SKIP_INSTALL_ALL)
    install(FILES ${MINIZIP_HDR} DESTINATION "${INSTALL_INC_DIR}")
endif()

IF(NOT PRIVATE_EXTERNAL)
	SET(CMAKE_CONFIGURATION_TYPES "Debug;RelWithDebInfo" CACHE STRING "" FORCE)
	SET(CMAKE_INSTALL_PREFIX "C:/minizip-static" CACHE PATH "" FORCE)
ENDIF()