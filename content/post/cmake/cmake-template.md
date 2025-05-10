---
title: cmake编写模板
description: 难用的cmake语法，但是记下来随时copy它！
date: 2025-03-04
categories:
    - cmake
tags:
    - cmake
    - c++
    - vcpkg
---

1. cmake 输出相关信息

    ``` cmake
    message(STATUS "CMake version: ${CMAKE_VERSION}")
    message(STATUS "CMake C compiler: ${CMAKE_C_COMPILER}")
    message(STATUS "CMake CXX compiler: ${CMAKE_CXX_COMPILER}")
    message(STATUS "Build directory: ${PROJECT_BINARY_DIR}")
    message(STATUS "Build type: ${CMAKE_BUILD_TYPE}")
    ```

2. 判断是否被当作subproject

    ```cmake
    # Determine if is built as a subproject (using add_subdirectory)
    # or if it is the master project.
    if (NOT DEFINED XXX_MASTER_PROJECT)
      set(XXX_MASTER_PROJECT OFF)
      if (CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_SOURCE_DIR)
        set(XXX_MASTER_PROJECT ON)
      endif ()
      message(STATUS "Is build as a master project: ${XXX_MASTER_PROJECT}")
    endif ()
    ```

3. 生成compile_commands.json以供clangd使用

   windows下需要使用Ninja，如果使用vscode则在设置里把`cmake.generator`设置为`Ninja`

    ```cmake
    # generate compile_commands.json
    set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
    ```

4. cmake设置rpath，unix下搜索同目录下的so文件

    ```cmake
    set(CMAKE_SKIP_BUILD_RPATH  FALSE) # 加入rpath
    set(CMAKE_BUILD_WITH_INSTALL_RPATH TRUE) # 表示编译的时候是否使用CMAKE_INSTALL_RPATH作为rpath路径
    set(CMAKE_INSTALL_RPATH "\${ORIGIN}") # 设置rpath安装时的路径，${ORIGIN}代表运行时当前目录
    ```

    参考[How to set rpath origin in cmake?](https://stackoverflow.com/questions/58360502/how-to-set-rpath-origin-in-cmake)
5. cmake设置输出目录，解决windows下命令行生成目录不正确但是IDE生成目录正确的问题
   **一定要放在project命令后面**

    ```cmake
    set(XXX_OUTPUT_DIR ${PROJECT_BINARY_DIR}/bin)
    set(XXX_OUTPUT_LIB_DIR ${PROJECT_BINARY_DIR}/lib)
    set(XXX_OUTPUT_PDB_DIR ${PROJECT_BINARY_DIR}/pdb)
    set(XXX_PLUGINS_OUTPUT_DIR ${XXX_OUTPUT_DIR}/plugins)
    # do not set cmake internal variables in functions(not work...)
    # difference between windows and linux, see https://cmake.org/cmake/help/latest/manual/cmake-buildsystem.7.html#runtime-output-artifacts
    set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${XXX_OUTPUT_DIR})
    set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${XXX_OUTPUT_DIR})
    set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${XXX_OUTPUT_LIB_DIR})
    set(CMAKE_PDB_OUTPUT_DIRECTORY ${XXX_OUTPUT_PDB_DIR})
    set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_${CMAKE_BUILD_TYPE} ${XXX_OUTPUT_DIR})
    set(CMAKE_LIBRARY_OUTPUT_DIRECTORY_${CMAKE_BUILD_TYPE} ${XXX_OUTPUT_DIR})
    set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY_${CMAKE_BUILD_TYPE} ${XXX_OUTPUT_LIB_DIR})
    set(CMAKE_PDB_OUTPUT_DIRECTORY_${CMAKE_BUILD_TYPE} ${XXX_OUTPUT_PDB_DIR})
    ```

6. cmake开启fPIC

    ```cmake
    if (NOT DEFINED CMAKE_POSITION_INDEPENDENT_CODE)
        # Otherwise we can't link .so libs with .a libs
        set(CMAKE_POSITION_INDEPENDENT_CODE ON)
    endif()
    ```

7. cmake统一符号可见性，默认隐藏

    ```cmake
    if (XXX_MASTER_PROJECT AND NOT DEFINED CMAKE_CXX_VISIBILITY_PRESET)
        set(CMAKE_CXX_VISIBILITY_PRESET hidden CACHE STRING "Preset for the export of private symbols")
        set_property(CACHE CMAKE_CXX_VISIBILITY_PRESET PROPERTY STRINGS hidden default)
    endif ()

    if (XXX_MASTER_PROJECT AND NOT DEFINED CMAKE_VISIBILITY_INLINES_HIDDEN)
        set(CMAKE_VISIBILITY_INLINES_HIDDEN ON CACHE BOOL "Whether to add a compile flag to hide symbols of inline functions")
    endif ()
    ```

8. 集成vcpkg toolchain

    ```cmake
    if(DEFINED ENV{VCPKG_ROOT} AND NOT DEFINED CMAKE_TOOLCHAIN_FILE)
        message(STATUS "VCPKG_ROOT: $ENV{VCPKG_ROOT}")
        set(CMAKE_TOOLCHAIN_FILE "$ENV{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake" CACHE STRING "")
    endif()
    ```

9. 集成vcpkg manifest
    **一定要放在project命令之前**

    ``` cmake
    if (XXX_RUN_TEST)
        list(APPEND VCPKG_MANIFEST_FEATURES "tests")
    endif()
    if (XXX_BUILD_EXAMPLES)
        list(APPEND VCPKG_MANIFEST_FEATURES "examples")
    endif()
    if (XXX_BUILD_BENCHMARKS)
        list(APPEND VCPKG_MANIFEST_FEATURES "benchmarks")
    endif()
    ```

10. ctest集成gtest

    ``` cmake
    enable_testing()
    file(GLOB_RECURSE source CONFIGURE_DEPENDS *.h *.cpp)
    find_package(GTest CONFIG REQUIRED)
    add_executable(test_main ${source})
    target_link_libraries(test_main PRIVATE GTest::gtest)

    include(GoogleTest)
    gtest_discover_tests(test_main WORKING_DIRECTORY ${XXX_OUTPUT_DIR})
    ```
