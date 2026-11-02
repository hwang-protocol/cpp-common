# cpp-common: 各项目共用的 header-only 库
#   线程池、有界阻塞队列、无锁 SPSC 环形缓冲、延迟直方图、SQLite RAII 封装
#
# 两个 target:
#   cpp_common::common  只依赖线程库
#   cpp_common::sqlite  额外链接 SQLite (只有用到 common/sqlite.h 的项目才需要)
#
# 其它项目这样引入 (固定到某个 commit，见各项目的 CMakeLists.txt):
#   FetchContent_Declare(cpp_common GIT_REPOSITORY ... GIT_TAG <commit SHA>)
#   FetchContent_MakeAvailable(cpp_common)
cmake_minimum_required(VERSION 3.24)
project(cpp_common CXX)

find_package(Threads REQUIRED)

add_library(common INTERFACE)
add_library(cpp_common::common ALIAS common)
target_include_directories(common INTERFACE ${CMAKE_CURRENT_SOURCE_DIR}/include)
target_compile_features(common INTERFACE cxx_std_23)
target_link_libraries(common INTERFACE Threads::Threads)

# SQLite 是可选的：找不到就不提供 cpp_common::sqlite
find_package(SQLite3 QUIET)
if(SQLite3_FOUND)
    add_library(common_sqlite INTERFACE)
    add_library(cpp_common::sqlite ALIAS common_sqlite)
    if(TARGET SQLite3::SQLite3)              # CMake >= 4.3 的新名字
        target_link_libraries(common_sqlite INTERFACE common SQLite3::SQLite3)
    else()
        target_link_libraries(common_sqlite INTERFACE common SQLite::SQLite3)
    endif()
endif()

# 只有单独构建本仓库时才编译测试；被别的项目引入时不编译
if(PROJECT_IS_TOP_LEVEL)
    set(CMAKE_CXX_STANDARD 23)
    set(CMAKE_CXX_STANDARD_REQUIRED ON)
    set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
    add_compile_options(-Wall -Wextra -Wpedantic)
    enable_testing()
    if(NOT SQLite3_FOUND)
        message(FATAL_ERROR "tests need SQLite3 (macOS 自带；Ubuntu: apt install libsqlite3-dev)")
    endif()
    add_executable(common_tests tests/common_tests.cpp)
    target_link_libraries(common_tests PRIVATE common_sqlite)
    add_test(NAME common_tests COMMAND common_tests)
endif()
