file(GLOB sources *.cpp roaring.c)
file(GLOB headers *.h *.hpp *.hh *.tcc)

list(REMOVE_ITEM sources Bifrost.cpp)

add_library(bifrost_static STATIC ${sources} ${headers})
add_library(bifrost_dynamic SHARED ${sources} ${headers})

add_executable(Bifrost Bifrost.cpp)
set_property(SOURCE BlockedBloomFilter.cpp APPEND_STRING PROPERTY COMPILE_FLAGS " -funroll-loops")

set_target_properties(bifrost_static PROPERTIES OUTPUT_NAME "bifrost")
set_target_properties(bifrost_dynamic PROPERTIES OUTPUT_NAME "bifrost")

target_include_directories(bifrost_static PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})
target_include_directories(bifrost_dynamic PUBLIC
    $<INSTALL_INTERFACE:include>
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
    )
target_include_directories(Bifrost PRIVATE ${CMAKE_CURRENT_SOURCE_DIR})

target_compile_features(bifrost_static PRIVATE cxx_std_11 c_std_11)
target_compile_features(bifrost_dynamic PRIVATE cxx_std_11 c_std_11)
target_compile_features(Bifrost PRIVATE cxx_std_11 c_std_11)

target_compile_definitions(bifrost_dynamic PUBLIC MAX_KMER_SIZE=${MAX_KMER_SIZE} XXH_NAMESPACE=BIFROST_HASH_)
target_compile_definitions(bifrost_static PUBLIC MAX_KMER_SIZE=${MAX_KMER_SIZE} XXH_NAMESPACE=BIFROST_HASH_)
target_compile_options(bifrost_static PUBLIC -fPIC)

if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    # Make sure to add trailing space
    target_compile_options(bifrost_static PUBLIC $<$<CONFIG:Profile>:-pg>)
    target_compile_options(bifrost_dynamic PUBLIC $<$<CONFIG:Profile>:-pg>)
    target_link_libraries(bifrost_static PUBLIC $<$<CONFIG:Profile>:-pg>)
    target_link_libraries(bifrost_dynamic PUBLIC $<$<CONFIG:Profile>:-pg>)
else()
    if("${CONFIG}" STREQUAL "Profile" OR "${CMAKE_BUILD_TYPE}" STREQUAL "Profile")
        message(FATAL_ERROR "Profiling is only supported with GCC")
    endif()
endif()

if(COMPILATION_ARCH MATCHES "OFF")
	message("Disabling native architecture compilation (including AVX2)")
else()
	message("Compilation architecture: ${COMPILATION_ARCH}")
    target_compile_options(bifrost_static PUBLIC -march=${COMPILATION_ARCH})
    target_compile_options(bifrost_dynamic PUBLIC -march=${COMPILATION_ARCH})
endif()

if(ENABLE_AVX2 MATCHES "OFF")
	message("Disabling AVX2 instructions")
    target_compile_options(bifrost_static PUBLIC -mno-avx2)
    target_compile_options(bifrost_dynamic PUBLIC -mno-avx2)
endif(ENABLE_AVX2 MATCHES "OFF")

find_package(Threads REQUIRED)
target_link_libraries(bifrost_static PUBLIC ${CMAKE_THREAD_LIBS_INIT})
target_link_libraries(bifrost_dynamic PUBLIC ${CMAKE_THREAD_LIBS_INIT})

find_package(ZLIB REQUIRED)

if (ZLIB_FOUND)
    target_link_libraries(bifrost_static PUBLIC ${ZLIB_LIBRARIES})
    target_link_libraries(bifrost_dynamic PUBLIC ${ZLIB_LIBRARIES})

    target_include_directories(bifrost_static PUBLIC ${ZLIB_INCLUDE_DIRS})
    target_include_directories(bifrost_dynamic PUBLIC ${ZLIB_INCLUDE_DIRS})
else()
    message(FATAL_ERROR "zlib not found. Required for to output files")
endif(ZLIB_FOUND)

target_link_libraries(Bifrost PUBLIC bifrost_static)

install(TARGETS Bifrost DESTINATION bin)
install(TARGETS bifrost_dynamic DESTINATION lib)
install(TARGETS bifrost_static DESTINATION lib)
install(FILES ${headers} DESTINATION include/bifrost)
install(FILES xxhash.c DESTINATION include/bifrost)
