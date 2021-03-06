cmake_minimum_required(VERSION 2.6)

PROJECT(snappy)

set(SNAPPY_MAJOR 1)
set(SNAPPY_MINOR 1)
set(SNAPPY_PATCHLEVEL 1)

############ configure ################

INCLUDE (CheckIncludeFiles)
INCLUDE (CheckFunctionExists)
INCLUDE (TestBigEndian)

CHECK_INCLUDE_FILES(stdint.h HAVE_STDINT_H)
CHECK_INCLUDE_FILES(stddef.h HAVE_STDDEF_H)
CHECK_INCLUDE_FILES(byteswap.h HAVE_BYTESWAP_H)
CHECK_INCLUDE_FILES(dlfcn.h HAVE_DLFCN_H)
CHECK_INCLUDE_FILES(inttypes.h HAVE_INTTYPES_H)
CHECK_INCLUDE_FILES(memory.h HAVE_MEMORY_H)
CHECK_INCLUDE_FILES(stdlib.h HAVE_STDLIB_H)
CHECK_INCLUDE_FILES(strings.h HAVE_STRINGS_H)
CHECK_INCLUDE_FILES(string.h HAVE_STRING_H)
CHECK_INCLUDE_FILES(sys/byteswap.h HAVE_SYS_BYTESWAP_H)
CHECK_INCLUDE_FILES(sys/endian.h HAVE_SYS_ENDIAN_H)
CHECK_INCLUDE_FILES(sys/mman.h HAVE_SYS_MMAN_H)
CHECK_INCLUDE_FILES(sys/mman.h HAVE_FUNC_MMAP)
CHECK_INCLUDE_FILES(sys/resource.h HAVE_SYS_RESOURCE_H)
CHECK_INCLUDE_FILES(sys/stat.h HAVE_SYS_STAT_H)
CHECK_INCLUDE_FILES(sys/time.h HAVE_SYS_TIME_H)
CHECK_INCLUDE_FILES(sys/types.h HAVE_SYS_TYPES_H)
CHECK_INCLUDE_FILES(unistd.h HAVE_UNISTD_H)
CHECK_INCLUDE_FILES(windows.h HAVE_WINDOWS_H)
CHECK_INCLUDE_FILES(sys/uio.h HAVE_SYS_UIO_H)

TEST_BIG_ENDIAN(WORDS_BIGENDIAN)

set(TRY_COMPILE_CODE "if(__builtin_expect(1<2, 1)) {}")
CONFIGURE_FILE(${PROJECT_SOURCE_DIR}/cmake/try-compile.c ${CMAKE_BINARY_DIR}/check-builtin-expect.c)
TRY_COMPILE(HAVE_BUILTIN_EXPECT ${CMAKE_BINARY_DIR} 
            ${CMAKE_BINARY_DIR}/check-builtin-expect.c) 

set(TRY_COMPILE_CODE "int leading_zeros = __builtin_ctzll(4)")
CONFIGURE_FILE(${PROJECT_SOURCE_DIR}/cmake/try-compile.c ${CMAKE_BINARY_DIR}/check-builtin-ctz.c)
TRY_COMPILE(HAVE_BUILTIN_CTZ ${CMAKE_BINARY_DIR} 
            ${CMAKE_BINARY_DIR}/check-builtin-ctz.c) 
            
find_package (ZLIB)
if(ZLIB_FOUND)
    set(HAVE_LIBZ 1)
    set(TEST_INCLUDES ${TEST_INCLUDES}  ${ZLIB_INCLUDE_DIRS})
endif()

# TODO: port discovery of the following libraries from configure.ac:
# CHECK_EXT_COMPRESSION_LIB([lzo2], [lzo1x_1_15_compress])
# CHECK_EXT_COMPRESSION_LIB([lzf], [lzf_compress])
# CHECK_EXT_COMPRESSION_LIB([fastlz], [fastlz_compress])
# CHECK_EXT_COMPRESSION_LIB([quicklz], [qlz_compress])
# GFLAGS
# GTEST

CONFIGURE_FILE(cmake/config.h.cmake ${CMAKE_BINARY_DIR}/config.h)
CONFIGURE_FILE(cmake/snappy-stubs-public.h.cmake ${PROJECT_SOURCE_DIR}/snappy-stubs-public.h @ONLY)

if(MSVC)
    ADD_DEFINITIONS(-DWIN32)
endif()

ADD_DEFINITIONS(-DHAVE_CONFIG_H)

############## create libraries ###############

INCLUDE_DIRECTORIES(${PROJECT_SOURCE_DIR} ${CMAKE_BINARY_DIR})

ADD_LIBRARY(snappy_static STATIC
    snappy-c.cc  snappy-sinksource.cc  snappy-stubs-internal.cc  snappy.cc)

ADD_LIBRARY(snappy SHARED
    snappy-c.cc  snappy-sinksource.cc  snappy-stubs-internal.cc  snappy.cc)

if(MSVC)
    set_target_properties(snappy PROPERTIES COMPILE_DEFINITIONS "SNAPPY_DLL")
    set_target_properties(snappy_static PROPERTIES COMPILE_DEFINITIONS "SNAPPY_STATIC_LIB")
endif()

############ compile test ######################

INCLUDE_DIRECTORIES(${TEST_INCLUDES})
ADD_EXECUTABLE(snappy_unittest snappy_unittest.cc snappy-test.cc)
target_link_libraries(snappy_unittest snappy ${ZLIB_LIBRARIES} ${LZO2_LIBRARIES})

enable_testing()
add_test(NAME snappy_test WORKING_DIRECTORY ${PROJECT_SOURCE_DIR} COMMAND snappy_unittest)

########### install #########################

INSTALL(FILES snappy.h snappy-stubs-public.h
        DESTINATION include)
INSTALL(TARGETS snappy snappy_static
        RUNTIME DESTINATION bin 
        LIBRARY DESTINATION lib 
        ARCHIVE DESTINATION lib)
