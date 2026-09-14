# Copyright(c) 2020 livoxtech limited.

cmake_minimum_required(VERSION 3.14)

include(cmake/version.cmake)

# C言語用のインターフェース生成に対応するため LANGUAGES に C を追加
project(livox_ros_driver2 VERSION ${LIVOX_ROS_DRIVER2_VERSION} LANGUAGES C CXX)
message(STATUS "${PROJECT_NAME} version: ${LIVOX_ROS_DRIVER2_VERSION}")

#---------------------------------------------------------------------------------------
# Compiler Config
#---------------------------------------------------------------------------------------
if(NOT CMAKE_C_STANDARD)
  set(CMAKE_C_STANDARD 99)
endif()

if(NOT CMAKE_CXX_STANDARD)
  set(CMAKE_CXX_STANDARD 17)
endif()

list(INSERT CMAKE_MODULE_PATH 0 "${PROJECT_SOURCE_DIR}/cmake/modules")

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic -Wno-unused-parameter)
endif()

# Add ROS Version MACRO
add_definitions(-DBUILDING_ROS2)

#---------------------------------------------------------------------------------------
# Find Dependencies
#---------------------------------------------------------------------------------------
# PCL/VTK 依存関係のエラーを回避するため、先に MPI を検出
find_package(MPI QUIET)

find_package(ament_cmake_auto REQUIRED)
ament_auto_find_build_dependencies()

find_package(PCL REQUIRED)
find_package(std_msgs REQUIRED)
find_package(builtin_interfaces REQUIRED)
find_package(rosidl_default_generators REQUIRED)

# Check APR
find_package(PkgConfig)
pkg_check_modules(APR apr-1)
if (APR_FOUND)
  message(STATUS "APR Include Dirs: ${APR_INCLUDE_DIRS}")
  message(STATUS "APR Libraries: ${APR_LIBRARIES}")
endif (APR_FOUND)

# Make sure the livox_lidar_sdk_shared library is installed
find_library(LIVOX_LIDAR_SDK_LIBRARY liblivox_lidar_sdk_shared.so /usr/local/lib REQUIRED)

find_path(LIVOX_LIDAR_SDK_INCLUDE_DIR
  NAMES "livox_lidar_api.h" "livox_lidar_def.h"
  REQUIRED
)

# PCL Library settings
link_directories(${PCL_LIBRARY_DIRS})
add_definitions(${PCL_DEFINITIONS})

#---------------------------------------------------------------------------------------
# Generate Custom Messages (ROS 2)
#---------------------------------------------------------------------------------------
set(LIVOX_INTERFACES livox_interfaces2)
rosidl_generate_interfaces(${LIVOX_INTERFACES}
  "msg/CustomPoint.msg"
  "msg/CustomMsg.msg"
  DEPENDENCIES builtin_interfaces std_msgs
  LIBRARY_NAME ${PROJECT_NAME}
)

#---------------------------------------------------------------------------------------
# Create Target Library
#---------------------------------------------------------------------------------------
ament_auto_add_library(${PROJECT_NAME} SHARED
  src/livox_ros_driver2.cpp
  src/lddc.cpp
  src/driver_node.cpp
  src/lds.cpp
  src/lds_lidar.cpp

  src/comm/comm.cpp
  src/comm/ldq.cpp
  src/comm/semaphore.cpp
  src/comm/lidar_imu_data_queue.cpp
  src/comm/cache_index.cpp
  src/comm/pub_handler.cpp

  src/parse_cfg_file/parse_cfg_file.cpp
  src/parse_cfg_file/parse_livox_lidar_cfg.cpp

  src/call_back/lidar_common_callback.cpp
  src/call_back/livox_lidar_callback.cpp
)

# Link custom message typesupport for ROS 2 Humble
rosidl_get_typesupport_target(cpp_typesupport_target
  ${LIVOX_INTERFACES} "rosidl_typesupport_cpp")

# Include Directories
target_include_directories(${PROJECT_NAME} PUBLIC
  ${PCL_INCLUDE_DIRS}
  ${APR_INCLUDE_DIRS}
  ${LIVOX_LIDAR_SDK_INCLUDE_DIR}
  3rdparty
  src
)

# Link Libraries
target_link_libraries(${PROJECT_NAME}
  "${cpp_typesupport_target}"
  ${LIVOX_LIDAR_SDK_LIBRARY}
  ${PPT_LIBRARY}
  ${Boost_LIBRARY}
  ${PCL_LIBRARIES}
  ${APR_LIBRARIES}
)

# Register Component Node
rclcpp_components_register_node(${PROJECT_NAME}
  PLUGIN "livox_ros::DriverNode"
  EXECUTABLE ${PROJECT_NAME}_node
)

#---------------------------------------------------------------------------------------
# Testing & Installation
#---------------------------------------------------------------------------------------
if(BUILD_TESTING)
  find_package(ament_lint_auto REQUIRED)
  ament_lint_auto_find_test_dependencies()
endif()

ament_auto_package(INSTALL_TO_SHARE
  config
  launch
)