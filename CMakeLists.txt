cmake_minimum_required(VERSION 3.20)
project(Jungle VERSION 1.0.0 LANGUAGES CXX)

include(CTest) # automatically defines BUILD_TESTING option

option(WITH_CONAN "Use dependences provide by Conan" OFF)
option(SNAPPY_OPTION "Use Snappy Compression" OFF)
option(CODE_COVERAGE "Enable Coverage" OFF)
option(ADDRESS_SANITIZER "Enable address sanitizer" OFF)
option(THREAD_SANITIZER "Enable thread sanitizer" OFF)
option(LEAK_SANITIZER "Enable leak sanitizer" OFF)
option(BUILD_EXAMPLES "Build examples" ON)
option(DETACH_LOGGER "Build libsimplelogger.a" ON)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# === Build type (default: RelWithDebInfo, O2) ===========
if(NOT CMAKE_BUILD_TYPE)
    set(BUILD_TYPE_OPTIONS
        "Choose the type of build, "
        "options are: Debug Release RelWithDebInfo MinSizeRel.")
    set(CMAKE_BUILD_TYPE "RelWithDebInfo"
        CACHE ${BUILD_TYPE_OPTIONS} FORCE)
    message(STATUS "Build type is not given, use default.")
endif()
message(STATUS "Build type: " ${CMAKE_BUILD_TYPE})


set(ROOT_SRC ${PROJECT_SOURCE_DIR}/src)

if(CODE_COVERAGE)
    set(CMAKE_BUILD_TYPE "Debug")
    include(cmake/CodeCoverage.cmake)
    message(STATUS "---- CODE COVERAGE DETECTION MODE ----")
endif()

# Libraries

if(WITH_CONAN)
    find_package(forestdb REQUIRED)
    find_package(ZLIB REQUIRED)
    set(FORESTDB forestdb::forestdb)
    set(LIBZ ZLIB::ZLIB)
else()
    set(FORESTDB ${CMAKE_CURRENT_SOURCE_DIR}/third_party/forestdb/build/libforestdb.a)
    set(LIBZ z)
endif()

if(SNAPPY_OPTION)
    if(WITH_CONAN)
        find_package(Snappy REQUIRED)
        set(LIBSNAPPY Snappy::snappy)
    else()
        include(cmake/FindSnappy.cmake)
        if(SNAPPY_FOUND)
            set(LIBSNAPPY ${SNAPPY_LIBRARIES})
        else()
            message(FATAL_ERROR "Can't find snappy, "
                "if you want to build without snappy set SNAPPY_OPTION=OFF")
        endif(SNAPPY_FOUND)

    endif(WITH_CONAN)
    message(STATUS "Snappy option is enabled.")
    add_compile_definitions(SNAPPY_AVAILABLE=1)
endif()

set(LIBDL dl)
set(LIBPTHREAD pthread)

# Includes
include_directories(BEFORE
    src
    include
    tools
    tests
    $<$<NOT:$<BOOL:${WITH_CONAN}>>:${CMAKE_CURRENT_SOURCE_DIR}/third_party/forestdb/include>
)

add_compile_options(
    -g -Wall -Wno-pessimizing-move
    $<$<CXX_COMPILER_ID:Clang>:-Wno-vla-cxx-extension>
    $<$<NOT:$<BOOL:${APPLE}>>:-pthread>
)

if(CODE_COVERAGE)
    APPEND_COVERAGE_COMPILER_FLAGS()
    #set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-exceptions")
    #set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-inline")
    #set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D_NO_EXCEPTION")
    set(COVERAGE_EXCLUDES
        '*third_party/*'
        '*usr/*'
        '*tests/*'
        '*tools/*'
        '*src/simple_thread_pool.h'
        '*src/logger.*'
    )
endif()


# === SANITIZER ===

if(ADDRESS_SANITIZER)
    add_compile_options(-fsanitize=address -fuse-ld=gold)
    add_link_options(-fsanitize=address -fuse-ld=gold)
    message(STATUS "---- ADDRESS SANITIZER IS ON ----")
endif()

if(THREAD_SANITIZER)
    add_compile_options(-fsanitize=thread)
    add_link_options(-fsanitize=thread)
    message(STATUS "---- THREAD SANITIZER IS ON ----")
endif()

if(LEAK_SANITIZER)
    add_compile_options(-fsanitize=leak)
    add_link_options(-fsanitize=leak)
    message(STATUS "---- LEAK SANITIZER IS ON ----")
endif()



# === Program flags ===

if(TESTSUITE_NO_COLOR GREATER 0)
    add_compile_definitions(TESTSUITE_NO_COLOR=1)
    message(STATUS "---- MONOCOLOR TESTSUITE ----")
endif()

if(LOGGER_NO_BACKTRACE GREATER 0)
    add_compile_definitions(LOGGER_NO_BACKTRACE=1)
    message(STATUS "---- NO BACKTRACE BY LOGGER ----")
endif()

file(COPY ${PROJECT_SOURCE_DIR}/scripts/runtests.sh
    DESTINATION ${CMAKE_CURRENT_BINARY_DIR})

# === CUSTOM LOGGER ===

if(LOGGER_PATH)
    set(LOGGER_CC_FILE "${LOGGER_PATH}/logger.cc")
    set(LOGGER_HEADER_FILE "${LOGGER_PATH}/logger.h")
elseif(NOT LOGGER_CC_FILE)
    set(LOGGER_CC_FILE "${ROOT_SRC}/logger.cc")
    set(LOGGER_HEADER_FILE "${ROOT_SRC}/logger.h")
endif()
add_compile_definitions(LOGGER_H=${LOGGER_HEADER_FILE})
message(STATUS "Simple logger cc file: ${LOGGER_CC_FILE}")
message(STATUS "Simple logger header file: ${LOGGER_HEADER_FILE}")


# === Source files ===================
set(LOGGER_SRC ${LOGGER_CC_FILE})

if(DETACH_LOGGER)
    message(STATUS "---- DETACH LOGGER ----")
    # set(LIBSIMPLELOGGER "${CMAKE_CURRENT_BINARY_DIR}/libsimplelogger.a")
    # The proper/safe way is to pass the name of the target
    set(LIBSIMPLELOGGER simplelogger_lib)

    add_library(simplelogger_lib ${LOGGER_SRC})
    set_target_properties(simplelogger_lib PROPERTIES OUTPUT_NAME simplelogger
        CLEAN_DIRECT_OUTPUT 1)
    get_filename_component(ABSOLUTE_LOGGER_HEADER_FILE ${LOGGER_HEADER_FILE}
                           REALPATH BASE_DIR "${CMAKE_BINARY_DIR}")
    get_filename_component(LOGGER_HEADER_DIRECTORY ${ABSOLUTE_LOGGER_HEADER_FILE}
                           DIRECTORY)
    message(STATUS "Simple logger header directory: ${LOGGER_HEADER_DIRECTORY}")
    set_target_properties(simplelogger_lib PROPERTIES INCLUDE_DIRECTORIES
                          "${LOGGER_HEADER_DIRECTORY}")


else()
    set(LOGGER_SRC_TO_CORE ${LOGGER_SRC})

endif()

set(JUNGLE_CORE
    ${ROOT_SRC}/avltree.cc
    ${ROOT_SRC}/bloomfilter.cc
    ${ROOT_SRC}/cmd_handler.cc
    ${ROOT_SRC}/compactor.cc
    ${ROOT_SRC}/crc32.cc
    ${ROOT_SRC}/db_config.cc
    ${ROOT_SRC}/db_group.cc
    ${ROOT_SRC}/db_manifest.cc
    ${ROOT_SRC}/db_mgr.cc
    ${ROOT_SRC}/fileops_posix.cc
    ${ROOT_SRC}/fileops_directio.cc
    ${ROOT_SRC}/flusher.cc
    ${ROOT_SRC}/global_batch.cc
    ${ROOT_SRC}/internal_helper.cc
    ${ROOT_SRC}/iterator.cc
    ${ROOT_SRC}/jungle.cc
    ${ROOT_SRC}/jungle_builder.cc
    ${ROOT_SRC}/log_file.cc
    ${ROOT_SRC}/log_iterator.cc
    ${ROOT_SRC}/log_manifest.cc
    ${ROOT_SRC}/log_mgr.cc
    ${ROOT_SRC}/log_reclaimer.cc
    ${ROOT_SRC}/memtable.cc
    ${ROOT_SRC}/memtable_iterator.cc
    ${ROOT_SRC}/murmurhash3.cc
    ${ROOT_SRC}/sampler.cc
    ${ROOT_SRC}/skiplist.cc
    ${ROOT_SRC}/table_append.cc
    ${ROOT_SRC}/table_compact_condition.cc
    ${ROOT_SRC}/table_compaction.cc
    ${ROOT_SRC}/table_file.cc
    ${ROOT_SRC}/table_file_compaction.cc
    ${ROOT_SRC}/table_file_iterator.cc
    ${ROOT_SRC}/table_helper.cc
    ${ROOT_SRC}/table_iterator.cc
    ${ROOT_SRC}/table_lookup_booster.cc
    ${ROOT_SRC}/table_manifest.cc
    ${ROOT_SRC}/table_mgr.cc
    ${ROOT_SRC}/table_set_batch.cc
    ${ROOT_SRC}/table_split.cc
    ${ROOT_SRC}/table_writer.cc
    ${ROOT_SRC}/worker_mgr.cc
    ${LOGGER_SRC_TO_CORE}
    ${OPEN_MEMSTREAM}
)
#add_library(JUNGLE_CORE_OBJ OBJECT ${JUNGLE_CORE})

# Note: static libraries MUST be located in front of all shared libraries.
set(JUNGLE_DEPS
    ${FORESTDB}
    ${LIBSNAPPY}
    ${LIBPTHREAD}
    ${LIBDL})

add_library(static_lib ${JUNGLE_CORE})
target_include_directories(static_lib PUBLIC "${PROJECT_SOURCE_DIR}/include")
target_link_libraries(static_lib ${JUNGLE_DEPS})
set_target_properties(static_lib PROPERTIES OUTPUT_NAME jungle
    CLEAN_DIRECT_OUTPUT 1)
if(DETACH_LOGGER GREATER 0)
    add_dependencies(static_lib simplelogger_lib)
endif()

if(BUILD_EXAMPLES)
    add_subdirectory(examples)
endif()

if(BUILD_TESTING)
    add_subdirectory(tests)
endif()

add_subdirectory(tools)

if(CODE_COVERAGE GREATER 0)
    SETUP_TARGET_FOR_COVERAGE(
        NAME jungle_cov
        EXECUTABLE ./runtests.sh
        DEPENDENCIES keyvalue_test
        crc32_test
        fileops_test
        fileops_directio_test
        memtable_test
        table_test
        basic_op_test
        sync_and_flush_test
        nearest_search_test
        seq_itr_test
        key_itr_test
        snapshot_test
        custom_cmp_test
        corruption_test
        compaction_test
        mt_test
        log_reclaim_test
        custom_mani_verifier
        level_extension_test
        table_lookup_booster_test
        compression_test
        atomic_batch_test
        builder_test
    )
endif()

install(DIRECTORY src/ DESTINATION include FILES_MATCHING PATTERN "*.h")
install(DIRECTORY include/ DESTINATION include)
install(FILES tools/mutable_table_mgr.h DESTINATION include/libjungle)
install(TARGETS static_lib simplelogger_lib DESTINATION lib ARCHIVE DESTINATION lib)
