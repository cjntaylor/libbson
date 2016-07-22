cmake_minimum_required(VERSION 3.0)

project(libbson)

if(NOT CMAKE_CONFIGURATION_TYPES AND NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Debug" CACHE STRING "Choose the type of build, \
        options are: Debug Release RelWithDebInfo MinSizeRel." FORCE
    )
endif()

set(BSON_LIBRARY_NAME bson)
set(BSON_MAJOR_VERSION 1)
set(BSON_MINOR_VERSION 3)
set(BSON_MICRO_VERSION 5)
set(BSON_VERSION 
    ${BSON_MAJOR_VERSION}.${BSON_MINOR_VERSION}.${BSON_MICRO_VERSION}
)
set(BSON_VERSION_SOVER ${BSON_MAJOR_VERSION})

option(BSON_TESTS_ENABLED "Enable testing" On)

option(BSON_BUILD_STATIC "Build static library" On)
option(BSON_BUILD_SHARED "Build shared library" On)
if(APPLE)
    option(BSON_BUILD_FRAMEWORK "Build OSX framework" On)
endif()

# Catch invalid configuration state (no library types)
if(NOT BSON_BUILD_SHARED AND NOT BSON_BUILD_STATIC AND 
   NOT BSON_BUILD_FRAMEWORK)
    message(FATAL_ERROR "No library set to build")
endif()

# Add the local cmake folder to the module search path
list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake)

# Make sure we're building out of source
include(EnsureOutOfSourceBuild)
ensure_out_of_source_build("\
    ${PROJECT_NAME} requires an out of source build. \
    Please create a separate build directory and run \
    'cmake /path/to/${PROJECT_NAME} [options]' there.\
")

set(BSON_API_VERSION 1.0)

include(CheckFunctionExists)
include(CheckIncludeFile)
include(CheckStructHasMember)
include(CheckSymbolExists)
include(TestBigEndian)

test_big_endian(BSON_BIG_ENDIAN)

set(BSON_EXPERIMENTAL_FEATURES 0)
set(BSON_PTHREAD_ONCE_INIT_NEEDS_BRACES 0)
set(BSON_HAVE_DECIMAL128 0)

#librt needed on linux for clock_gettime
find_package(RT)
if(RT_FOUND)
   #set required libraries for check_function_exists
   set(CMAKE_REQUIRED_LIBRARIES ${RT_LIBRARIES})
endif()

# See https://public.kitware.com/Bug/view.php?id=15659
check_symbol_exists(snprintf stdio.h BSON_HAVE_SNPRINTF)
if(NOT BSON_HAVE_SNPRINTF)
  set(BSON_HAVE_SNPRINTF 0)
else()
  set(BSON_HAVE_SNPRINTF 1)
endif()

check_function_exists(_set_output_format BSON_NEEDS_SET_OUTPUT_FORMAT)
if(NOT BSON_NEEDS_SET_OUTPUT_FORMAT)
  set(BSON_NEEDS_SET_OUTPUT_FORMAT 0)
endif ()

check_function_exists(reallocf BSON_HAVE_REALLOCF)
if(NOT BSON_HAVE_REALLOCF)
  set(BSON_HAVE_REALLOCF 0)
endif()

check_struct_has_member("struct timespec" tv_sec time.h BSON_HAVE_TIMESPEC)
if(NOT BSON_HAVE_TIMESPEC)
  message(STATUS "    no timespec struct")
  set(BSON_HAVE_TIMESPEC 0)
else()
  message(STATUS "    struct timespec found")
  set(BSON_HAVE_TIMESPEC 1)
endif()

if (MSVC)
   set(BSON_OS 2)
   set(BSON_HAVE_CLOCK_GETTIME 0)
   set(BSON_HAVE_STDBOOL_H 0)
   set(BSON_HAVE_STRNLEN 0)
   set(BSON_EXTRA_ALIGN 1)
   set(BSON_HAVE_SYSCALL_TID 0)

   find_package(Ws2_32 REQUIRED)
   list(APPEND BSON_LIBRARIES ${WS2_32_LIBRARIES})
else()
   find_package(Threads)
   list(APPEND BSON_LIBRARIES ${CMAKE_THREAD_LIBS_INIT})

   set(BSON_OS 1)
   check_function_exists(clock_gettime BSON_HAVE_CLOCK_GETTIME)
   if(NOT BSON_HAVE_CLOCK_GETTIME)
      set(BSON_HAVE_CLOCK_GETTIME 0)
   endif()
   check_function_exists(strnlen BSON_HAVE_STRNLEN)
   if(NOT BSON_HAVE_STRNLEN)
      set(BSON_HAVE_STRNLEN 0)
   endif()
   check_include_file(stdbool.h BSON_HAVE_STDBOOL_H)
   if(NOT BSON_HAVE_STDBOOL_H)
      set(BSON_HAVE_STDBOOL_H 0)
   endif ()
   set(BSON_EXTRA_ALIGN 1)
   check_function_exists(syscall BSON_HAVE_SYSCALL_TID)
   check_include_file(strings.h HAVE_STRINGS_H)
endif ()

set(BSON_HAVE_ATOMIC_32_ADD_AND_FETCH 1)
set(BSON_HAVE_ATOMIC_64_ADD_AND_FETCH 1)

if(BSON_BIG_ENDIAN)
   set(BSON_BYTE_ORDER 4321)
else()
   set(BSON_BYTE_ORDER 1234)
endif()

configure_file (
   ${CMAKE_CURRENT_SOURCE_DIR}/src/bson/bson-config.h.in
   ${CMAKE_CURRENT_BINARY_DIR}/bson-config.h
)

configure_file (
   ${CMAKE_CURRENT_SOURCE_DIR}/src/bson/bson-version.h.in
   ${CMAKE_CURRENT_BINARY_DIR}/bson-version.h
)

configure_file (
   ${CMAKE_CURRENT_SOURCE_DIR}/build/cmake/bson/bson-stdint.h
   ${CMAKE_CURRENT_BINARY_DIR}/bson-stdint.h
)

list(APPEND BSON_HEADERS
    ${CMAKE_CURRENT_BINARY_DIR}/bson-config.h
    ${CMAKE_CURRENT_BINARY_DIR}/bson-version.h
    ${CMAKE_CURRENT_BINARY_DIR}/bson-stdint.h
    src/bson/bcon.h
    src/bson/bson-atomic.h
    src/bson/bson-clock.h
    src/bson/bson-compat.h
    src/bson/bson-context.h
    src/bson/bson-endian.h
    src/bson/bson-error.h
    src/bson/bson.h
    src/bson/bson-iter.h
    src/bson/bson-json.h
    src/bson/bson-keys.h
    src/bson/bson-macros.h
    src/bson/bson-md5.h
    src/bson/bson-memory.h
    src/bson/bson-oid.h
    src/bson/bson-reader.h
    src/bson/bson-stdint-win32.h
    src/bson/bson-string.h
    src/bson/bson-types.h
    src/bson/bson-utf8.h
    src/bson/bson-value.h
    src/bson/bson-version-functions.h
    src/bson/bson-writer.h
)

list(APPEND BSON_SOURCES
    src/bson/bcon.c
    src/bson/bson.c
    src/bson/bson-atomic.c
    src/bson/bson-clock.c
    src/bson/bson-context.c
    src/bson/bson-error.c
    src/bson/bson-iso8601.c
    src/bson/bson-iter.c
    src/bson/bson-json.c
    src/bson/bson-keys.c
    src/bson/bson-md5.c
    src/bson/bson-memory.c
    src/bson/bson-oid.c
    src/bson/bson-reader.c
    src/bson/bson-string.c
    src/bson/bson-timegm.c
    src/bson/bson-utf8.c
    src/bson/bson-value.c
    src/bson/bson-version-functions.c
    src/bson/bson-writer.c
    src/yajl/yajl_alloc.c
    src/yajl/yajl_buf.c
    src/yajl/yajl.c
    src/yajl/yajl_encode.c
    src/yajl/yajl_gen.c
    src/yajl/yajl_lex.c
    src/yajl/yajl_parser.c
    src/yajl/yajl_tree.c
    src/yajl/yajl_version.c
)

if(WIN32)
    list(APPEND BSON_SOURCES 
        ${CMAKE_CURRENT_SOURCE_DIR}/build/cmake/libbson.def
    )
endif()

list(APPEND BSON_INCLUDE
    PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src/bson>
    PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>
    PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
)

list(APPEND BSON_DEFINES
    PRIVATE BSON_COMPILATION
    PRIVATE $<$<CONFIG:Debug>:DEBUG>
    PRIVATE $<$<NOT:$<CONFIG:Debug>>:NDEBUG>
)

# Set the install path per platform
if(WIN32)
    # Use C:\Program Files\${PROJECT_NAME}\<version> under windows (since there
    # isn't a common "lib" folder)
    set(BSON_INSTALL_PATH ${PROJECT_NAME}/${BSON_VERSION})
    set(BSON_LIBRARY_INSTALL_PATH ${BSON_INSTALL_PATH})
    set(BSON_RUNTIME_INSTALL_PATH ${BSON_INSTALL_PATH})
    set(BSON_INCLUDE_INSTALL_PATH ${BSON_INSTALL_PATH}/include)
    set(BSON_PACKAGE_INSTALL_PATH cmake/${PROJECT_NAME})
else()
    set(BSON_LIBRARY_INSTALL_PATH lib)
    # Use library architecture if it is set (multiarch support)
    if(CMAKE_LIBRARY_ARCHITECTURE)
        set(BSON_LIBRARY_INSTALL_PATH
            "${BSON_LIBRARY_INSTALL_PATH}/${CMAKE_LIBRARY_ARCHITECTURE}"
        )
    endif()
    set(BSON_RUNTIME_INSTALL_PATH bin)
    set(BSON_INCLUDE_INSTALL_PATH include/${PROJECT_NAME}-${BSON_VERSION})
    set(BSON_PACKAGE_INSTALL_PATH 
        ${BSON_LIBRARY_INSTALL_PATH}/cmake/${PROJECT_NAME}
    )
endif()

list(APPEND BSON_INCLUDE 
    PUBLIC $<INSTALL_INTERFACE:${BSON_INCLUDE_INSTALL_PATH}>
)

# Generate output tree
if(NOT CMAKE_OUTPUT_DIRECTORY)
    set(CMAKE_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR})
endif()
set(CMAKE_INCLUDE_OUTPUT_DIRECTORY 
    ${CMAKE_OUTPUT_DIRECTORY}/${BSON_INCLUDE_INSTALL_PATH}
)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY 
    ${CMAKE_OUTPUT_DIRECTORY}/${BSON_LIBRARY_INSTALL_PATH}
)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY 
    ${CMAKE_OUTPUT_DIRECTORY}/${BSON_RUNTIME_INSTALL_PATH}
)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY 
    ${CMAKE_LIBRARY_OUTPUT_DIRECTORY}
)
set(CMAKE_PACKAGE_OUTPUT_DIRECTORY 
    ${CMAKE_OUTPUT_DIRECTORY}/${BSON_PACKAGE_INSTALL_PATH}
)

# Provide an uninstall target
if(NOT TARGET uninstall)
    add_custom_target(uninstall
        COMMAND ${CMAKE_COMMAND}
        ARGS -P ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake
    )
endif()

# Dependencies go here

# Install public headers to the output tree
file(INSTALL ${BSON_HEADERS} DESTINATION ${CMAKE_INCLUDE_OUTPUT_DIRECTORY})

# Generate a shared library
if(BSON_BUILD_SHARED)
    add_library(${PROJECT_NAME}-shared SHARED 
        ${BSON_HEADERS}
        ${BSON_SOURCES}
    )
    # USING_BSON_SHARED here DEPENDS on the ordering of defines in uv.h
    target_compile_definitions(${PROJECT_NAME}-shared ${BSON_DEFINES})
    target_include_directories(${PROJECT_NAME}-shared ${BSON_INCLUDE})
    target_link_libraries(${PROJECT_NAME}-shared ${BSON_LIBRARIES})
    set_target_properties(${PROJECT_NAME}-shared PROPERTIES
        OUTPUT_NAME ${BSON_LIBRARY_NAME}
        LIBRARY_OUTPUT_DIRECTORY_DEBUG ${CMAKE_LIBRARY_OUTPUT_DIRECTORY}
        LIBRARY_OUTPUT_DIRECTORY_RELEASE ${CMAKE_LIBRARY_OUTPUT_DIRECTORY}
        LIBRARY_OUTPUT_DIRECTORY_MINSIZEREL ${CMAKE_LIBRARY_OUTPUT_DIRECTORY}
        LIBRARY_OUTPUT_DIRECTORY_RELWITHDEBINFO 
            ${CMAKE_LIBRARY_OUTPUT_DIRECTORY}
        DEBUG_POSTFIX _d
        VERSION ${BSON_VERSION}
        SOVERSION ${BSON_VERSION_SOVER}
        POSITION_INDEPENDENT_CODE True
    )
    list(APPEND BSON_EXPORT_TARGETS ${PROJECT_NAME}-shared)
    if(NOT BSON_BUILT_LIBRARY)
        set(BSON_BUILT_LIBRARY ${PROJECT_NAME}-shared)
    endif()
endif()

# Generate an OSX framework
if(BSON_BUILD_FRAMEWORK)
    add_library(${PROJECT_NAME}-framework SHARED
        ${BSON_HEADERS}
        ${BSON_SOURCES}
    )
    target_compile_definitions(${PROJECT_NAME}-framework ${BSON_DEFINES})
    target_include_directories(${PROJECT_NAME}-framework ${BSON_INCLUDE})
    target_link_libraries(${PROJECT_NAME}-framework ${BSON_LIBRARIES})
    set_target_properties(${PROJECT_NAME}-framework PROPERTIES
        OUTPUT_NAME ${PROJECT_NAME}$<$<CONFIG:Debug>:_d>
        LIBRARY_OUTPUT_DIRECTORY_DEBUG ${CMAKE_LIBRARY_OUTPUT_DIRECTORY}
        LIBRARY_OUTPUT_DIRECTORY_RELEASE ${CMAKE_LIBRARY_OUTPUT_DIRECTORY}
        LIBRARY_OUTPUT_DIRECTORY_MINSIZEREL ${CMAKE_LIBRARY_OUTPUT_DIRECTORY}
        LIBRARY_OUTPUT_DIRECTORY_RELWITHDEBINFO 
            ${CMAKE_LIBRARY_OUTPUT_DIRECTORY}
        VERSION ${BSON_VERSION}
        SOVERSION ${BSON_VERSION_SOVER}
        POSITION_INDEPENDENT_CODE True
        FRAMEWORK True
        FRAMEWORK_VERSION ${BSON_VERSION}
        PUBLIC_HEADER "${BSON_HEADERS}"
    )
    list(APPEND BSON_EXPORT_TARGETS_LOCAL ${PROJECT_NAME}-framework)
endif()

# Generate a static library
if(BSON_BUILD_STATIC)
    add_library(${PROJECT_NAME}-static STATIC
        ${BSON_HEADERS}
        ${BSON_SOURCES}
    )
    target_compile_definitions(${PROJECT_NAME}-static ${BSON_DEFINES})
    target_include_directories(${PROJECT_NAME}-static ${BSON_INCLUDE})
    target_link_libraries(${PROJECT_NAME}-static ${BSON_LIBRARIES})
    set_target_properties(${PROJECT_NAME}-static PROPERTIES
        OUTPUT_NAME ${BSON_LIBRARY_NAME}
        ARCHIVE_OUTPUT_DIRECTORY_DEBUG ${CMAKE_ARCHIVE_OUTPUT_DIRECTORY}
        ARCHIVE_OUTPUT_DIRECTORY_RELEASE ${CMAKE_ARCHIVE_OUTPUT_DIRECTORY}
        ARCHIVE_OUTPUT_DIRECTORY_MINSIZEREL ${CMAKE_ARCHIVE_OUTPUT_DIRECTORY}
        ARCHIVE_OUTPUT_DIRECTORY_RELWITHDEBINFO 
            ${CMAKE_ARCHIVE_OUTPUT_DIRECTORY}
        DEBUG_POSTFIX _d
    )
    list(APPEND BSON_EXPORT_TARGETS ${PROJECT_NAME}-static)
    if(NOT BSON_BUILT_LIBRARY)
        set(BSON_BUILT_LIBRARY ${PROJECT_NAME}-static)
    endif()
endif()

# Include installed export targets in local targets
list(APPEND BSON_EXPORT_TARGETS_LOCAL ${BSON_EXPORT_TARGETS})

# Install uv libraries and headers
install(TARGETS ${BSON_EXPORT_TARGETS}
    EXPORT ${PROJECT_NAME}
    LIBRARY DESTINATION ${BSON_LIBRARY_INSTALL_PATH}
    ARCHIVE DESTINATION ${BSON_LIBRARY_INSTALL_PATH}
)
install(FILES ${BSON_HEADERS} DESTINATION ${BSON_INCLUDE_INSTALL_PATH})

# Generate in-tree package configuration
include(CMakePackageConfigHelpers)
write_basic_package_version_file(
    ${CMAKE_PACKAGE_OUTPUT_DIRECTORY}/${PROJECT_NAME}ConfigVersion.cmake
    VERSION ${BSON_VERSION}
    COMPATIBILITY SameMajorVersion
)
export(TARGETS ${BSON_EXPORT_TARGETS_LOCAL}
    FILE ${CMAKE_PACKAGE_OUTPUT_DIRECTORY}/${PROJECT_NAME}Config.cmake
)

# Generate installed package configuration
install(
    FILES ${CMAKE_PACKAGE_OUTPUT_DIRECTORY}/${PROJECT_NAME}ConfigVersion.cmake
    DESTINATION ${BSON_PACKAGE_INSTALL_PATH}
)
install(EXPORT ${PROJECT_NAME} 
    DESTINATION ${BSON_PACKAGE_INSTALL_PATH}
    FILE ${PROJECT_NAME}Config.cmake
)

# Configure uninstall target
configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/cmake_uninstall.cmake.in
    ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake
    IMMEDIATE @ONLY
)
configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/uninstall_libbson.cmake.in
    ${CMAKE_OUTPUT_DIRECTORY}/uninstall/uninstall_libbson.cmake
    IMMEDIATE @ONLY
)

if(BSON_TESTS_ENABLED)
    enable_testing()

    list(APPEND BSON_TEST_SOURCES
        tests/TestSuite.c
        tests/TestSuite.h
        tests/test-libbson.c
        tests/test-atomic.c
        tests/test-bson.c
        tests/test-endian.c
        tests/test-clock.c
        tests/test-error.c
        tests/test-iso8601.c
        tests/test-iter.c
        tests/test-json.c
        tests/test-oid.c
        tests/test-reader.c
        tests/test-string.c
        tests/test-utf8.c
        tests/test-value.c
        tests/test-version.c
        tests/test-writer.c
        tests/test-bcon-basic.c
        tests/test-bcon-extract.c
        tests/json-test.c
        tests/json-test.c
        tests/json-test.h
    )

    add_executable(test-libbson ${BSON_TEST_SOURCES})
    target_link_libraries(test-libbson ${BSON_BUILT_LIBRARY})
    target_compile_definitions(test-libbson
        PRIVATE BINARY_DIR=\"${CMAKE_CURRENT_SOURCE_DIR}/tests/binary\"
        PRIVATE JSON_DIR=\"${CMAKE_CURRENT_SOURCE_DIR}/tests/json\"
    )
    set_target_properties(test-libbson PROPERTIES
        RUNTIME_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
        POSITION_INDEPENDENT_CODE True
    )
    add_test(NAME test-libbson COMMAND test-libbson)
endif()