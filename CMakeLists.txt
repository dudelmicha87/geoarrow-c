message(STATUS "Building using CMake version: ${CMAKE_VERSION}")
cmake_minimum_required(VERSION 3.14)

if(NOT DEFINED CMAKE_C_STANDARD)
  set(CMAKE_C_STANDARD 99)
endif()

set(GEOARROW_VERSION "0.2.0-SNAPSHOT")
string(REGEX MATCH "^[0-9]+\\.[0-9]+\\.[0-9]+" GEOARROW_BASE_VERSION
             "${GEOARROW_VERSION}")
project(geoarrow VERSION "${GEOARROW_BASE_VERSION}")

set(GEOARROW_VERSION_MAJOR "${geoarrow_VERSION_MAJOR}")
set(GEOARROW_VERSION_MINOR "${geoarrow_VERSION_MINOR}")
set(GEOARROW_VERSION_PATCH "${geoarrow_VERSION_PATCH}")

option(GEOARROW_BUILD_TESTS "Build tests" OFF)
option(GEOARROW_CODE_COVERAGE "Enable coverage reporting" OFF)
option(GEOARROW_USE_FAST_FLOAT "Use fast_float for numeric value parsing" ON)
option(GEOARROW_USE_RYU "Use ryu for numeric value printing" ON)
option(GEOARROW_NAMESPACE "A prefix for exported symbols" OFF)
option(GEOARROW_ARROW_STATIC "Use a statically-linked Arrow C++ build when linking tests"
       OFF)
option(GEOARROW_BUNDLE "Use ON to just bundle the headers/sources" OFF)

if(GEOARROW_NAMESPACE)
  set(GEOARROW_NAMESPACE_DEFINE "#define GEOARROW_NAMESPACE ${GEOARROW_NAMESPACE}")
else()
  set(GEOARROW_NAMESPACE_DEFINE "// #define GEOARROW_NAMESPACE YourNamespaceHere")
endif()

add_library(geoarrow_coverage_config INTERFACE)
install(TARGETS geoarrow_coverage_config
        DESTINATION lib
        EXPORT geoarrow-exports)

if(GEOARROW_CODE_COVERAGE)
  target_compile_options(geoarrow_coverage_config INTERFACE -O0 -g --coverage)
  target_link_options(geoarrow_coverage_config INTERFACE --coverage)
endif()

if(GEOARROW_USE_FAST_FLOAT)
  if(NOT DEFINED CMAKE_CXX_STANDARD)
    set(CMAKE_CXX_STANDARD 11)
  endif()

  set(GEOARROW_DOUBLE_PARSE_SOURCE src/geoarrow/double_parse_fast_float.cc)
  set(GEOARROW_USE_FAST_FLOAT_DEFINE "#define GEOARROW_USE_FAST_FLOAT 1")
else()
  set(GEOARROW_DOUBLE_PARSE_SOURCE src/geoarrow/double_parse_std.c)
  set(GEOARROW_USE_FAST_FLOAT_DEFINE "#define GEOARROW_USE_FAST_FLOAT 0")
endif()

if(GEOARROW_USE_RYU)
  set(GEOARROW_DOUBLE_PRINT_SOURCE src/geoarrow/double_print.c src/geoarrow/ryu/d2s.c)
  set(GEOARROW_USE_RYU_DEFINE "#define GEOARROW_USE_RYU 1")
else()
  set(GEOARROW_DOUBLE_PRINT_SOURCE src/geoarrow/double_print.c)
  set(GEOARROW_USE_RYU_DEFINE "#define GEOARROW_USE_RYU 0")
endif()

configure_file(src/geoarrow/geoarrow_config.h.in generated/geoarrow/geoarrow_config.h)

if(NOT TARGET nanoarrow::nanoarrow
   AND NOT GEOARROW_BUNDLE
   OR GEOARROW_BUILD_TESTS)
  set(GEOARROW_NANOARROW_TARGET geoarrow_nanoarrow)
  add_library(geoarrow_nanoarrow STATIC EXCLUDE_FROM_ALL src/vendor/nanoarrow/nanoarrow.c)
  target_include_directories(geoarrow_nanoarrow
                             PUBLIC
                             PUBLIC "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src/vendor>"
  )
  target_compile_definitions(geoarrow_nanoarrow
                             PUBLIC "$<$<CONFIG:Debug>:NANOARROW_DEBUG>"
                                    NANOARROW_NAMESPACE=${GEOARROW_NAMESPACE})
  install(TARGETS geoarrow_nanoarrow
          DESTINATION lib
          EXPORT geoarrow-exports)
else()
  set(GEOARROW_NANOARROW_TARGET nanoarrow::nanoarrow)
endif()

set(GEOARROW_SOURCES
    src/geoarrow/schema.c
    src/geoarrow/schema_view.c
    src/geoarrow/metadata.c
    src/geoarrow/kernel.c
    src/geoarrow/builder.c
    src/geoarrow/array_view.c
    src/geoarrow/util.c
    src/geoarrow/visitor.c
    src/geoarrow/geometry.c
    src/geoarrow/native_writer.c
    src/geoarrow/wkb_reader.c
    src/geoarrow/wkb_writer.c
    src/geoarrow/wkt_reader.c
    src/geoarrow/wkt_writer.c
    src/geoarrow/array_reader.c
    src/geoarrow/array_writer.c)

set(GEOARROW_PUBLIC_HEADERS
    ${CMAKE_CURRENT_BINARY_DIR}/generated/geoarrow/geoarrow_config.h
    src/geoarrow/geoarrow_type.h src/geoarrow/geoarrow.h
    src/geoarrow/geoarrow_type_inline.h)

if(GEOARROW_BUNDLE)
  # Bundle sources + headers, and if building tests, add a target the tests can link against
  file(REMOVE_RECURSE ${CMAKE_BINARY_DIR}/amalgamation)
  file(MAKE_DIRECTORY ${CMAKE_BINARY_DIR}/amalgamation)
  file(MAKE_DIRECTORY ${CMAKE_BINARY_DIR}/amalgamation/geoarrow)

  set(GEOARROW_H_TEMP ${CMAKE_BINARY_DIR}/amalgamation/geoarrow/geoarrow.h)
  file(WRITE ${GEOARROW_H_TEMP} "")

  foreach(SRC_FILE ${GEOARROW_PUBLIC_HEADERS})
    file(READ ${SRC_FILE} SRC_FILE_CONTENTS)
    file(APPEND ${GEOARROW_H_TEMP} "${SRC_FILE_CONTENTS}")
  endforeach()

  set(GEOARROW_C_TEMP ${CMAKE_BINARY_DIR}/amalgamation/geoarrow/geoarrow.c)

  foreach(SRC_FILE ${GEOARROW_SOURCES} src/geoarrow/double_print.c
                   src/geoarrow/double_parse_std.c)
    file(READ ${SRC_FILE} SRC_FILE_CONTENTS)
    file(APPEND ${GEOARROW_C_TEMP} "${SRC_FILE_CONTENTS}")
  endforeach()

  set(GEOARROW_HPP_TEMP ${CMAKE_BINARY_DIR}/amalgamation/geoarrow/geoarrow.hpp)
  file(WRITE ${GEOARROW_HPP_TEMP} "")
  foreach(SRC_FILE
          exception.hpp
          internal.hpp
          geometry_data_type.hpp
          array_reader.hpp
          array_writer.hpp
          array_util.hpp
          wkb_util.hpp
          geometry_type_traits.hpp)
    file(READ "src/geoarrow/hpp/${SRC_FILE}" SRC_FILE_CONTENTS)
    file(APPEND ${GEOARROW_HPP_TEMP} "${SRC_FILE_CONTENTS}")
  endforeach()

  # Update includes for concatenated sources
  foreach(SRC_FILE ${GEOARROW_HPP_TEMP} ${GEOARROW_H_TEMP} ${GEOARROW_C_TEMP})
    file(READ ${SRC_FILE} SRC_FILE_CONTENTS)
    string(REGEX REPLACE "#include \"geoarrow/[a-z_.]+\"" "" SRC_FILE_CONTENTS
                         "${SRC_FILE_CONTENTS}")
    string(REGEX REPLACE "#include \"geoarrow/hpp/[a-z_.]+\"" "" SRC_FILE_CONTENTS
                         "${SRC_FILE_CONTENTS}")
    file(WRITE ${SRC_FILE} "${SRC_FILE_CONTENTS}")
  endforeach()

  # Make sure geoarrow.h is included from geoarrow.c and geoarrow.hpp
  foreach(SRC_FILE ${GEOARROW_HPP_TEMP} ${GEOARROW_C_TEMP})
    file(READ ${SRC_FILE} SRC_FILE_CONTENTS)
    file(WRITE ${SRC_FILE} "#include \"geoarrow/geoarrow.h\"\n")
    file(APPEND ${SRC_FILE} "${SRC_FILE_CONTENTS}")
  endforeach()

  # "install" means copy the bundled sources to the specified dir. Just
  # geoarrow.c, geoarrow.h, and geoarrow.hpp for now.
  install(FILES ${GEOARROW_HPP_TEMP} ${GEOARROW_H_TEMP} ${GEOARROW_C_TEMP}
          DESTINATION ".")

  # If building tests, add the geoarrow target so we can test against it
  if(GEOARROW_BUILD_TESTS)
    # Make sure wkx_testing.hpp is also copied
    file(READ src/geoarrow/wkx_testing.hpp SRC_FILE_CONTENTS)
    file(WRITE ${CMAKE_BINARY_DIR}/amalgamation/geoarrow/wkx_testing.hpp
         "${SRC_FILE_CONTENTS}")

    add_library(geoarrow ${GEOARROW_C_TEMP})

    target_include_directories(geoarrow
                               PUBLIC "$<BUILD_INTERFACE:${CMAKE_BINARY_DIR}/amalgamation>"
                                      "$<INSTALL_INTERFACE:include>")
    target_compile_definitions(geoarrow PUBLIC "$<$<CONFIG:Debug>:GEOARROW_DEBUG>")
    target_link_libraries(geoarrow
                          PRIVATE ${GEOARROW_NANOARROW_TARGET}
                          PUBLIC geoarrow_coverage_config)
  endif()
else()
  # Normal CMake build
  add_library(geoarrow SHARED ${GEOARROW_SOURCES} ${GEOARROW_DOUBLE_PARSE_SOURCE}
                       ${GEOARROW_DOUBLE_PRINT_SOURCE})

  target_include_directories(geoarrow
                             PUBLIC "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>"
                                    "$<INSTALL_INTERFACE:include>")
  target_include_directories(geoarrow
                             PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/generated>
  )
  target_compile_definitions(geoarrow PUBLIC "$<$<CONFIG:Debug>:GEOARROW_DEBUG>")
  target_link_libraries(geoarrow
                        PRIVATE ${GEOARROW_NANOARROW_TARGET}
                        PUBLIC geoarrow_coverage_config)

  install(DIRECTORY src/geoarrow/hpp
          DESTINATION include/geoarrow
          FILES_MATCHING
          PATTERN "*.hpp")
  install(FILES src/geoarrow/geoarrow.h src/geoarrow/geoarrow_type.h
                src/geoarrow/geoarrow_type_inline.h src/geoarrow/geoarrow.hpp
          DESTINATION include/geoarrow)
  install(FILES ${CMAKE_CURRENT_BINARY_DIR}/generated/geoarrow/geoarrow_config.h
          DESTINATION include/geoarrow)
endif()

if(TARGET geoarrow)
  install(TARGETS geoarrow
          DESTINATION lib
          EXPORT geoarrow-exports)

  if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    if(CMAKE_C_COMPILER_ID STREQUAL "GNU")
      set(GEOARROW_EXTRA_WARNINGS
          -Wall
          -Werror
          -Wextra
          -Wpedantic
          -Wno-type-limits
          -Wmaybe-uninitialized
          -Wunused-result
          -Wconversion
          -Wno-sign-conversion)
    elseif(CMAKE_C_COMPILER_ID STREQUAL "AppleClang" OR CMAKE_C_COMPILER_ID STREQUAL
                                                        "Clang")
      set(GEOARROW_EXTRA_WARNINGS
          -Wall
          -Werror
          -Wextra
          -Wpedantic
          -Wdocumentation
          -Wconversion
          -Wno-sign-conversion)

    endif()
  endif()
endif()

# If building tests, ensure the tests also are compiled with these warnings.
# If not (but still in debug), still compile geoarrow sources with these warnings.
if(GEOARROW_BUILD_TESTS AND GEOARROW_EXTRA_WARNINGS)
  target_compile_options(geoarrow PUBLIC ${GEOARROW_EXTRA_WARNINGS})
elseif(GEOARROW_EXTRA_WARNINGS)
  target_compile_options(geoarrow PRIVATE ${GEOARROW_EXTRA_WARNINGS})
endif()

if(GEOARROW_BUILD_TESTS)
  set(MEMORYCHECK_COMMAND_OPTIONS
      "--leak-check=full --suppressions=${CMAKE_CURRENT_LIST_DIR}/valgrind.supp --error-exitcode=1"
  )
  include(CTest)
  include(FetchContent)

  find_package(Arrow REQUIRED)

  if(${ARROW_VERSION} VERSION_GREATER_EQUAL "10.0.0")
    set(CMAKE_CXX_STANDARD 17)
  else()
    set(CMAKE_CXX_STANDARD 11)
  endif()

  set(CMAKE_CXX_STANDARD_REQUIRED ON)

  if(${CMAKE_VERSION} VERSION_GREATER_EQUAL "3.23")
    cmake_policy(SET CMP0135 NEW)
  endif()

  # Give caller the option to link a static version of Arrow C++
  if(GEOARROW_ARROW_STATIC)
    set(GEOARROW_ARROW_TARGET arrow_static)
  else()
    set(GEOARROW_ARROW_TARGET arrow_shared)
  endif()

  # Use an old version of googletest if we have to to support gcc 4.8
  if(NOT CMAKE_CXX_COMPILER_ID STREQUAL "GNU" OR CMAKE_CXX_COMPILER_VERSION
                                                 VERSION_GREATER_EQUAL "5.0.0")
    fetchcontent_declare(googletest
                         URL https://github.com/google/googletest/archive/release-1.11.0.zip
                         URL_HASH SHA256=353571c2440176ded91c2de6d6cd88ddd41401d14692ec1f99e35d013feda55a
    )
  else()
    fetchcontent_declare(googletest
                         URL https://github.com/google/googletest/archive/release-1.10.0.zip
                         URL_HASH SHA256=94c634d499558a76fa649edb13721dce6e98fb1e7018dfaeba3cd7a083945e91
    )
  endif()

  fetchcontent_makeavailable(googletest)

  enable_testing()

  add_executable(geoarrow_type_inline_test src/geoarrow/geoarrow_type_inline_test.cc)
  add_executable(builder_test src/geoarrow/builder_test.cc)
  add_executable(array_view_test src/geoarrow/array_view_test.cc)
  add_executable(schema_test src/geoarrow/schema_test.cc)
  add_executable(schema_view_test src/geoarrow/schema_view_test.cc)
  add_executable(metadata_test src/geoarrow/metadata_test.cc)
  add_executable(geometry_test src/geoarrow/geometry_test.cc)
  add_executable(kernel_test src/geoarrow/kernel_test.cc)
  add_executable(visitor_test src/geoarrow/visitor_test.cc)
  add_executable(util_test src/geoarrow/util_test.cc)
  add_executable(native_writer_test src/geoarrow/native_writer_test.cc)
  add_executable(wkb_reader_test src/geoarrow/wkb_reader_test.cc)
  add_executable(wkb_writer_test src/geoarrow/wkb_writer_test.cc)
  add_executable(wkt_reader_test src/geoarrow/wkt_reader_test.cc)
  add_executable(wkt_writer_test src/geoarrow/wkt_writer_test.cc)
  add_executable(array_reader_test src/geoarrow/array_reader_test.cc)
  add_executable(array_writer_test src/geoarrow/array_writer_test.cc)
  add_executable(wkx_files_test src/geoarrow/wkx_files_test.cc)

  add_executable(hpp_array_writer_test src/geoarrow/hpp/array_writer_test.cc)
  add_executable(hpp_geometry_data_type_test src/geoarrow/hpp/geometry_data_type_test.cc)
  add_executable(hpp_arrow_extension_type_test
                 src/geoarrow/hpp/arrow_extension_type_test.cc)
  add_executable(hpp_array_util_test src/geoarrow/hpp/array_util_test.cc)
  add_executable(hpp_wkb_util_test src/geoarrow/hpp/wkb_util_test.cc)
  add_executable(hpp_geometry_type_traits_test
                 src/geoarrow/hpp/geometry_type_traits_test.cc)

  target_link_libraries(geoarrow_type_inline_test geoarrow gtest_main
                        ${GEOARROW_NANOARROW_TARGET})
  target_link_libraries(builder_test geoarrow gtest_main ${GEOARROW_NANOARROW_TARGET})
  target_link_libraries(array_view_test geoarrow gtest_main ${GEOARROW_NANOARROW_TARGET})
  target_link_libraries(schema_test geoarrow ${GEOARROW_ARROW_TARGET} gtest_main)
  target_link_libraries(schema_view_test geoarrow gtest_main ${GEOARROW_NANOARROW_TARGET})
  target_link_libraries(geometry_test geoarrow gtest_main ${GEOARROW_NANOARROW_TARGET})
  target_link_libraries(kernel_test geoarrow gtest_main ${GEOARROW_NANOARROW_TARGET})
  target_link_libraries(metadata_test geoarrow gtest_main ${GEOARROW_NANOARROW_TARGET})
  target_link_libraries(visitor_test geoarrow gtest_main)
  target_link_libraries(util_test geoarrow gtest_main)
  target_link_libraries(native_writer_test geoarrow gtest_main
                        ${GEOARROW_NANOARROW_TARGET})
  target_link_libraries(wkb_reader_test geoarrow gtest_main ${GEOARROW_NANOARROW_TARGET})
  target_link_libraries(wkb_writer_test geoarrow gtest_main ${GEOARROW_NANOARROW_TARGET})
  target_link_libraries(wkt_reader_test geoarrow gtest_main ${GEOARROW_NANOARROW_TARGET})
  target_link_libraries(wkt_writer_test geoarrow gtest_main ${GEOARROW_NANOARROW_TARGET})
  target_link_libraries(array_reader_test geoarrow gtest_main
                        ${GEOARROW_NANOARROW_TARGET})
  target_link_libraries(array_writer_test geoarrow gtest_main
                        ${GEOARROW_NANOARROW_TARGET})
  target_link_libraries(wkx_files_test geoarrow gtest_main ${GEOARROW_NANOARROW_TARGET})

  target_link_libraries(hpp_array_writer_test
                        geoarrow
                        gtest_main
                        gmock_main
                        ${GEOARROW_NANOARROW_TARGET})
  target_link_libraries(hpp_geometry_data_type_test geoarrow gtest_main
                        ${GEOARROW_NANOARROW_TARGET})
  target_link_libraries(hpp_arrow_extension_type_test geoarrow ${GEOARROW_ARROW_TARGET}
                        gtest_main)
  target_link_libraries(hpp_array_util_test
                        geoarrow
                        gtest_main
                        gmock_main
                        ${GEOARROW_NANOARROW_TARGET})
  target_link_libraries(hpp_wkb_util_test
                        geoarrow
                        gtest_main
                        gmock_main
                        ${GEOARROW_NANOARROW_TARGET})
  target_link_libraries(hpp_geometry_type_traits_test
                        geoarrow
                        gtest_main
                        gmock_main
                        ${GEOARROW_NANOARROW_TARGET})

  include(GoogleTest)
  gtest_discover_tests(geoarrow_type_inline_test)
  gtest_discover_tests(builder_test)
  gtest_discover_tests(array_view_test)
  gtest_discover_tests(schema_test)
  gtest_discover_tests(schema_view_test)
  gtest_discover_tests(geometry_test)
  gtest_discover_tests(kernel_test)
  gtest_discover_tests(metadata_test)
  gtest_discover_tests(visitor_test)
  gtest_discover_tests(util_test)
  gtest_discover_tests(native_writer_test)
  gtest_discover_tests(wkb_reader_test)
  gtest_discover_tests(wkb_writer_test)
  gtest_discover_tests(wkt_reader_test)
  gtest_discover_tests(wkt_writer_test)
  gtest_discover_tests(array_reader_test)
  gtest_discover_tests(array_writer_test)
  gtest_discover_tests(wkx_files_test)

  gtest_discover_tests(hpp_array_writer_test)
  gtest_discover_tests(hpp_geometry_data_type_test)
  gtest_discover_tests(hpp_arrow_extension_type_test)
  gtest_discover_tests(hpp_array_util_test)
  gtest_discover_tests(hpp_wkb_util_test)
  gtest_discover_tests(hpp_geometry_type_traits_test)
endif()

# Add the geoarrow alias target (won't exist in bundle mode, so skip it there)
if(TARGET geoarrow)
  add_library(geoarrow::geoarrow ALIAS geoarrow)

  # Generate package files for the build and install trees.
  include(CMakePackageConfigHelpers)
  include(GNUInstallDirs)

  foreach(tree_type BUILD INSTALL)
    if(tree_type STREQUAL "BUILD")
      set(install_location ".")
    else()
      set(install_location "${CMAKE_INSTALL_LIBDIR}/cmake/geoarrow")
    endif()

    set(build_location "${PROJECT_BINARY_DIR}/${install_location}")
    write_basic_package_version_file(
      "${build_location}/geoarrow-config-version.cmake"
      VERSION ${geoarrow_VERSION}
      # After 1.0.0, we can use `SameMajorVersion` here.
      COMPATIBILITY ExactVersion)
    configure_package_config_file("${CMAKE_CURRENT_LIST_DIR}/cmake/config.cmake.in"
                                  "${build_location}/geoarrow-config.cmake"
                                  INSTALL_DESTINATION "${install_location}")

    if(tree_type STREQUAL "BUILD")
      export(EXPORT geoarrow-exports
             FILE "${build_location}/geoarrow-targets.cmake"
             NAMESPACE geoarrow::)

    else()
      install(DIRECTORY "${build_location}/" DESTINATION "${install_location}")
      install(EXPORT geoarrow-exports
              DESTINATION "${install_location}"
              FILE "geoarrow-targets.cmake"
              NAMESPACE geoarrow::)
    endif()
  endforeach()
endif()
