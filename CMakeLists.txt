cmake_minimum_required(VERSION 3.10 FATAL_ERROR)
project(libdxfrw VERSION 0.6.3 LANGUAGES C CXX)

# Option: allow user to choose shared or static. Default ON to build shared.
option(BUILD_SHARED_LIBS "Build libdxfrw as a shared library" ON)

# Enable testing
enable_testing()

file(GLOB libdxfrw_sources src/*.cpp)
file(GLOB libdxfrw_headers include/*.h)
file(GLOB libdxfrw_intern_sources src/intern/*.cpp)

# Find iconv library
if(WIN32)
    include_directories(vs2013/packages/libiconv.1.14.0.11/build/native/include)
    link_directories(vs2013/packages/libiconv.1.14.0.11/build/native/lib)
    set(ICONV_LIBRARY iconv)
else()
    find_package(Iconv QUIET)
    if(Iconv_FOUND)
        include_directories(${Iconv_INCLUDE_DIRS})
        set(ICONV_LIBRARY ${Iconv_LIBRARIES})
    else()
        # Fallback: try to link against iconv directly
        set(ICONV_LIBRARY iconv)
    endif()
endif()

# Public include dir (install will copy headers)
include_directories(${CMAKE_SOURCE_DIR}/include)

# Add library (shared if BUILD_SHARED_LIBS=ON, static otherwise)
add_library(dxfrw ${libdxfrw_sources} ${libdxfrw_intern_sources})

# If building shared, define an export macro so headers can use it
if(BUILD_SHARED_LIBS)
  target_compile_definitions(dxfrw PRIVATE LIBDXFRW_BUILD_SHARED=1)
endif()

# Position independent code for shared libs (and modern practice)
set_target_properties(dxfrw PROPERTIES POSITION_INDEPENDENT_CODE ON)

# Hide symbols by default on GCC/Clang; export via macro header
if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
  target_compile_options(dxfrw PRIVATE -fvisibility=hidden)
endif()

# Link iconv (if present)
target_link_libraries(dxfrw PRIVATE ${ICONV_LIBRARY})

# Install rules
install(FILES ${libdxfrw_headers} DESTINATION include)

if(WIN32)
  install(TARGETS dxfrw
          RUNTIME DESTINATION bin
          ARCHIVE DESTINATION lib
          LIBRARY DESTINATION lib)
else()
  install(TARGETS dxfrw
          LIBRARY DESTINATION lib
          ARCHIVE DESTINATION lib)
endif()

# Helper function to add tests (keeps CMake concise)
function(add_dxfrw_test name src)
  add_executable(${name} ${src})
  target_include_directories(${name} PRIVATE ${CMAKE_SOURCE_DIR}/src ${CMAKE_SOURCE_DIR}/tests ${CMAKE_SOURCE_DIR}/include)
  target_link_libraries(${name} PRIVATE dxfrw ${ICONV_LIBRARY})
  add_test(NAME ${name} COMMAND ${name})
endfunction()

add_dxfrw_test(test_basic tests/test_basic.cpp)
add_dxfrw_test(test_entities tests/test_entities.cpp)
add_dxfrw_test(test_polylines tests/test_polylines.cpp)
add_dxfrw_test(test_text tests/test_text.cpp)
add_dxfrw_test(test_tables tests/test_tables.cpp)
add_dxfrw_test(test_blocks tests/test_blocks.cpp)
add_dxfrw_test(test_versions tests/test_versions.cpp)
add_dxfrw_test(test_errors tests/test_errors.cpp)

# Provide useful messages for users
message(STATUS "libdxfrw version: ${PROJECT_VERSION}")
if(BUILD_SHARED_LIBS)
    message(STATUS "Building as shared library")
else()
    message(STATUS "Building as static library")
endif()

# Platform-specific linker advice
if(APPLE)
  message(STATUS "On macOS you may need -liconv when linking if iconv isn't in libc")
endif()
