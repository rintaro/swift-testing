# This source file is part of the Swift.org open source project
#
# Copyright (c) 2024 Apple Inc. and the Swift project authors
# Licensed under Apache License v2.0 with Runtime Library Exception
#
# See http://swift.org/LICENSE.txt for license information
# See http://swift.org/CONTRIBUTORS.txt for Swift project authors

set(SwiftTesting_MACRO_PATH "<auto>" CACHE STRING "Path to SwiftTesting macro plugin, or '<auto>' for building it")

if(SwiftTesting_MACRO_PATH STREQUAL "<auto>")
  # Macros must be built for the build machine, not the host.
  include(ExternalProject)
  if(NOT SwiftTesting_MACRO_MAKE_PROGRAM)
    set(SwiftTesting_MACRO_MAKE_PROGRAM ${CMAKE_MAKE_PROGRAM})
  endif()
  if(NOT SwiftTesting_MACRO_Swift_COMPILER)
    set(SwiftTesting_MACRO_Swift_COMPILER ${CMAKE_Swift_COMPILER})
  endif()
  if(NOT SwiftTesting_MACRO_Swift_FLAGS)
    set(SwiftTesting_MACRO_Swift_FLAGS ${CMAKE_Swift_FLAGS})
    set(SwiftTesting_MACRO_SWIFT_FLAGS_RELEASE ${CMAKE_Swift_FLAGS_RELEASE})
    set(SwiftTesting_MACRO_SWIFT_FLAGS_RELWITHDEBINFO ${CMAKE_Swift_FLAGS_RELWITHDEBINFO})
  endif()
  if(NOT SwiftTesting_MACRO_AR)
    set(SwiftTesting_MACRO_AR ${CMAKE_AR})
  endif()
  if(NOT SwiftTesting_MACRO_RANLIB)
    set(SwiftTesting_MACRO_RANLIB ${CMAKE_RANLIB})
  endif()

  find_package(SwiftSyntax CONFIG GLOBAL)
  if(SwiftSyntax_FOUND)
    set(SwiftTesting_BuildMacrosAsExecutables NO)
  else()
    set(SwiftTesting_BuildMacrosAsExecutables YES)
  endif()

  ExternalProject_Add(TestingMacros
    PREFIX "tm"
    SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/TestingMacros"
    CMAKE_ARGS
      -DCMAKE_MAKE_PROGRAM=${SwiftTesting_MACRO_MAKE_PROGRAM}
      -DCMAKE_Swift_COMPILER=${SwiftTesting_MACRO_Swift_COMPILER}
      -DCMAKE_Swift_FLAGS=${SwiftTesting_MACRO_Swift_FLAGS}
      -DCMAKE_Swift_FLAGS_RELEASE=${SwiftTesting_MACRO_Swift_FLAGS_RELEASE}
      -DCMAKE_Swift_FLAGS_RELWITHDEBINFO=${SwiftTesting_MACRO_Swift_FLAGS_RELWITHDEBINFO}
      -DCMAKE_AR=${SwiftTesting_MACRO_AR}
      -DCMAKE_RANLIB=${SwiftTesting_MACRO_RANLIB}
      -DSwiftTesting_BuildMacrosAsExecutables=${SwiftTesting_BuildMacrosAsExecutables}
      -DSwiftSyntax_DIR=${SwiftSyntax_DIR})
  ExternalProject_Get_Property(TestingMacros BINARY_DIR)

  # Hardcode the known file names based on system name as a workaround since
  # TestingMacros uses `ExternalProject` and we cannot directly query the
  # properties of its targets here.
  if(NOT SwiftTesting_BuildMacrosAsExecutables)
    if(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
      set(SwiftTesting_MACRO "${BINARY_DIR}/libTestingMacros.dylib")
      install(PROGRAMS "${SwiftTesting_MACRO_LIBRARY}"
        DESTINATION lib/swift/host/plugins/testing)
    elseif(CMAKE_SYSTEM_NAME STREQUAL "Linux")
      set(SwiftTesting_MACRO "${BINARY_DIR}/libTestingMacros.so")
      install(PROGRAMS "${SwiftTesting_MACRO_LIBRARY}"
        DESTINATION lib/swift/host/plugins)
    elseif(CMAKE_SYSTEM_NAME STREQUAL "Windows")
      set(SwiftTesting_MACRO_LIBRARY "${BINARY_DIR}/TestingMacros.dll")
      # In Windows toolchain, TestingMacros.dll is injected from separate CMake build.
    else()
      message(FATAL_ERROR "Unable to determine the library name for TestingMacros based on system name: ${CMAKE_SYSTEM_NAME}")
    endif()
  else()
    if(CMAKE_SYSTEM_NAME STREQUAL "Windows")
      set(SwiftTesting_MACRO "${BINARY_DIR}/TestingMacros.exe")
    else()
      set(SwiftTesting_MACRO "${BINARY_DIR}/TestingMacros")
    endif()
  endif()
elseif(SwiftTesting_MACRO_PATH)
  add_custom_target(TestingMacros DEPENDS "${SwiftTesting_MACRO_PATH}")
  set(SwiftTesting_MACRO "${SwiftTesting_MACRO_PATH}")
else()
  add_custom_target(TestingMacros)
endif()

if(NOT SwiftTesting_MACRO)
  message(STATUS "TestingMacros: (none)")
elseif(SwiftTesting_MACRO)
  get_filename_component(macro_ext "${SwiftTesting_MACRO}" LAST_EXT)
  if(macro_ext STREQUAL ${CMAKE_SHARED_LIBRARY_SUFFIX})
    message(STATUS "TestingMacros: ${SwiftTesting_MACRO} (shared library)")
    add_compile_options("$<$<COMPILE_LANGUAGE:Swift>:SHELL:-load-plugin-library ${SwiftTesting_MACRO}>")
  else()
    message(STATUS "TestingMacros: ${SwiftTesting_MACRO} (executable)")
    add_compile_options("$<$<COMPILE_LANGUAGE:Swift>:SHELL:-load-plugin-exectuable ${SwiftTesting_MACRO}#TestingMacros>")
  endif()
endif()

include(AvailabilityDefinitions)
include(CompilerSettings)
add_subdirectory(_TestingInternals)
add_subdirectory(Testing)

