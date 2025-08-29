# ===== CMakeLists.txt (repo root) =====
cmake_minimum_required(VERSION 3.20)

# 프로젝트 이름은 레포 이름으로 자동 설정되게 시도하고, 실패하면 기본값 사용
get_filename_component(_repo_name "${CMAKE_SOURCE_DIR}" NAME)
if(NOT _repo_name)
  set(_repo_name "Project_with_CI")
endif()
project(${_repo_name} LANGUAGES C CXX)

# 컴파일 옵션
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)  # compile_commands.json

# 빌드 타입 기본값
if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
endif()

# include/ 경로 노출 (없어도 에러 안 나게 조건)
if(EXISTS "${CMAKE_SOURCE_DIR}/include")
  include_directories("${CMAKE_SOURCE_DIR}/include")
endif()

# ---- 라이브러리(옵션) ----
# src/lib/ 아래에 *.cpp가 있으면 정적 라이브러리 생성
file(GLOB_RECURSE LIB_SOURCES CONFIGURE_DEPENDS "${CMAKE_SOURCE_DIR}/src/lib/*.c" "${CMAKE_SOURCE_DIR}/src/lib/*.cpp")
if(LIB_SOURCES)
  add_library(${PROJECT_NAME}_lib STATIC ${LIB_SOURCES})
  target_include_directories(${PROJECT_NAME}_lib PUBLIC "${CMAKE_SOURCE_DIR}/include")
endif()

# ---- 실행파일 ----
# src/main.cpp 가 있으면 실행파일 생성
if(EXISTS "${CMAKE_SOURCE_DIR}/src/main.cpp")
  add_executable(${PROJECT_NAME} "${CMAKE_SOURCE_DIR}/src/main.cpp")
  if(TARGET ${PROJECT_NAME}_lib)
    target_link_libraries(${PROJECT_NAME} PRIVATE ${PROJECT_NAME}_lib)
  endif()
endif()

# ---- 유닛 테스트 ----
include(CTest)
option(BUILD_TESTING "Enable tests" ON)

if(BUILD_TESTING)
  # tests/*.cpp 있으면 GoogleTest 받아서 빌드
  file(GLOB TEST_SOURCES CONFIGURE_DEPENDS "${CMAKE_SOURCE_DIR}/tests/*.cpp")

  if(TEST_SOURCES)
    include(FetchContent)
    # GoogleTest 가져오기 (인터넷 필요)
    FetchContent_Declare(
      googletest
      URL https://github.com/google/googletest/archive/refs/tags/v1.14.0.zip
    )
    # Windows에서 MSVC 런타임 문제 피하기 옵션
    set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
    FetchContent_MakeAvailable(googletest)

    # 테스트 실행파일 생성
    foreach(test_file ${TEST_SOURCES})
      get_filename_component(test_name ${test_file} NAME_WE)
      add_executable(${test_name} ${test_file})
      target_link_libraries(${test_name} PRIVATE gtest gtest_main)
    endforeach()
  endif()
endif()