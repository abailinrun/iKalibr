# iKalibr: Unified Targetless Spatiotemporal Calibration Framework
# Copyright 2024, the School of Geodesy and Geomatics (SGG), Wuhan University, China
# https://github.com/Unsigned-Long/iKalibr.git
#
# Author: Shuolong Chen (shlchen@whu.edu.cn)
# GitHub: https://github.com/Unsigned-Long
#  ORCID: 0000-0002-5283-9057
#
# Purpose: See .h/.hpp file.
#
# Redistribution and use in source and binary forms, with or without
# modification, are permitted provided that the following conditions are met:
#
# * Redistributions of source code must retain the above copyright notice,
#   this list of conditions and the following disclaimer.
# * Redistributions in binary form must reproduce the above copyright notice,
#   this list of conditions and the following disclaimer in the documentation
#   and/or other materials provided with the distribution.
# * The names of its contributors can not be
#   used to endorse or promote products derived from this software without
#   specific prior written permission.
#
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
# AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
# IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
# ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT OWNER OR CONTRIBUTORS BE
# LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR
# CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF
# SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
# INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN
# CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE)
# ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE
# POSSIBILITY OF SUCH DAMAGE.

# Modified to support pure CMake build without ROS dependency

cmake_minimum_required(VERSION 3.16)
project(ikalibr)

# Modified: Option to build without ROS dependency
option(IKALIBR_NO_ROS "Build without ROS dependency (direct file input only)" OFF)

set(USE_CMAKE_UNITY_BUILD ON CACHE STRINGS "whether use 'CMAKE_UNITY_BUILD' in building")

if (${CMAKE_VERSION} VERSION_GREATER_EQUAL "3.16" AND USE_CMAKE_UNITY_BUILD)
    set(CMAKE_UNITY_BUILD ON)
    message(STATUS "use 'CMAKE_UNITY_BUILD' in building!")
else ()
    message(STATUS "do not use 'CMAKE_UNITY_BUILD' in building!")
endif ()

# C++17 standard
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_BUILD_TYPE "RELEASE")

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS}  -Wall   -O3")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall   -O3")
set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -march=native")
set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -march=native")

if (NOT DEFINED CMAKE_SUPPRESS_DEVELOPER_WARNINGS)
    set(CMAKE_SUPPRESS_DEVELOPER_WARNINGS 1 CACHE INTERNAL "No dev warnings")
endif ()

# Modified: Conditional ROS/catkin dependency
if (NOT IKALIBR_NO_ROS)
    message(STATUS "Building with ROS support")
    find_package(catkin REQUIRED COMPONENTS
            roscpp
            rospy
            std_msgs
            geometry_msgs
            sensor_msgs
            rosbag
            cv_bridge
            message_generation
            velodyne_msgs
            velodyne_pointcloud
    )

    # ROS message generation
    add_message_files(
            FILES
            AinsteinRadarTarget.msg
            AinsteinRadarTargetArray.msg
            SbgImuStatus.msg
            SbgImuData.msg
            LivoxCustomPoint.msg
            LivoxCustomMsg.msg
            AWR1843RadarScan.msg
            AWR1843RadarScanCustom.msg
            PropheseeEvent.msg
            PropheseeEventArray.msg
            DVSEvent.msg
            DVSEventArray.msg
    )

    generate_messages(
            DEPENDENCIES
            std_msgs
            geometry_msgs
    )
else ()
    message(STATUS "Building WITHOUT ROS support (IKALIBR_NO_ROS=ON)")
    add_definitions(-DIKALIBR_NO_ROS)
endif ()

###########
## Build ##
###########

set(tiny-viewer_DIR ${CMAKE_CURRENT_SOURCE_DIR}/thirdparty/ctraj/thirdparty/tiny-viewer-install/lib/cmake/tiny-viewer)
find_package(tiny-viewer)
set(ctraj_DIR ${CMAKE_CURRENT_SOURCE_DIR}/thirdparty/ctraj-install/lib/cmake/ctraj)
find_package(ctraj)
set(ufomap_DIR ${CMAKE_CURRENT_SOURCE_DIR}/thirdparty/ufomap-install/lib/cmake/ufomap)
find_package(ufomap)
set(ufomap_INCLUDE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/thirdparty/ufomap-install/include)
include_directories(${ufomap_INCLUDE_DIR})
set(veta_DIR ${CMAKE_CURRENT_SOURCE_DIR}/thirdparty/veta-install/lib/cmake/veta)
find_package(veta)
set(opengv_DIR ${CMAKE_CURRENT_SOURCE_DIR}/thirdparty/opengv-install/lib/cmake/opengv-1.0)
find_package(opengv)

find_package(OpenMP)
if (OPENMP_FOUND)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
endif ()

find_package(magic_enum)
# magic_enum works in range of [MAGIC_ENUM_RANGE_MIN, MAGIC_ENUM_RANGE_MAX]
add_definitions(-DMAGIC_ENUM_RANGE_MIN=0)
# 1024 = 2^10
add_definitions(-DMAGIC_ENUM_RANGE_MAX=1023)
find_package(yaml-cpp)
find_package(PCL REQUIRED)
find_package(OpenCV REQUIRED)
find_package(Eigen3 REQUIRED)
find_package(Ceres REQUIRED)

#define PCL_NO_PRECOMPILE [it's very important for Custom pcl Point type]
add_definitions(-DPCL_NO_PRECOMPILE)

aux_source_directory(${CMAKE_CURRENT_SOURCE_DIR}/src/calib CALIB_SRC_FILES)
aux_source_directory(${CMAKE_CURRENT_SOURCE_DIR}/src/solver SOLVER_SRC_FILES)
aux_source_directory(${CMAKE_CURRENT_SOURCE_DIR}/src/core CORE_SRC_FILES)
aux_source_directory(${CMAKE_CURRENT_SOURCE_DIR}/src/sensor SENSOR_SRC_FILES)
aux_source_directory(${CMAKE_CURRENT_SOURCE_DIR}/src/config CONFIG_SRC_FILES)
aux_source_directory(${CMAKE_CURRENT_SOURCE_DIR}/src/viewer VIEWER_SRC_FILES)
aux_source_directory(${CMAKE_CURRENT_SOURCE_DIR}/src/util UTIL_SRC_FILES)
aux_source_directory(${CMAKE_CURRENT_SOURCE_DIR}/src/factor FACTOR_SRC_FILES)

# Modified: Set up include directories and libraries based on ROS availability
if (NOT IKALIBR_NO_ROS)
    set(ROS_INCLUDE_DIRS ${catkin_INCLUDE_DIRS})
    set(ROS_LIBRARIES ${catkin_LIBRARIES})
else ()
    set(ROS_INCLUDE_DIRS "")
    set(ROS_LIBRARIES "")
endif ()

# Common include directories
set(COMMON_INCLUDE_DIRS
    ${CMAKE_CURRENT_SOURCE_DIR}/include
    ${PCL_INCLUDE_DIRS}
    ${OpenCV_INCLUDE_DIRS}
    ${EIGEN3_INCLUDE_DIR}
    ${CERES_INCLUDE_DIRS}
)

###################
# libikalibr_util #
###################
add_library(
        ${PROJECT_NAME}_util SHARED
        ${UTIL_SRC_FILES}
)

target_include_directories(
        ${PROJECT_NAME}_util PUBLIC
        ${ROS_INCLUDE_DIRS}
        ${COMMON_INCLUDE_DIRS}
)

target_link_libraries(
        ${PROJECT_NAME}_util PUBLIC
        ${ROS_LIBRARIES}
        ctraj
        veta
)

#####################
# libikalibr_sensor #
#####################
add_library(
        ${PROJECT_NAME}_sensor SHARED
        ${SENSOR_SRC_FILES}
)

target_include_directories(
        ${PROJECT_NAME}_sensor PUBLIC
        ${ROS_INCLUDE_DIRS}
        ${COMMON_INCLUDE_DIRS}
)

target_link_libraries(
        ${PROJECT_NAME}_sensor PUBLIC
        ${ROS_LIBRARIES}
        ctraj
        veta
)

#####################
# libikalibr_config #
#####################
add_library(
        ${PROJECT_NAME}_config SHARED
        ${CONFIG_SRC_FILES}
)

target_include_directories(
        ${PROJECT_NAME}_config PUBLIC
        ${ROS_INCLUDE_DIRS}
        ${COMMON_INCLUDE_DIRS}
)

target_link_libraries(
        ${PROJECT_NAME}_config PUBLIC
        ${ROS_LIBRARIES}
        ctraj
        veta
)

#####################
# libikalibr_factor #
#####################
add_library(
        ${PROJECT_NAME}_factor SHARED
        ${FACTOR_SRC_FILES}
)

target_include_directories(
        ${PROJECT_NAME}_factor PUBLIC
        ${ROS_INCLUDE_DIRS}
        ${COMMON_INCLUDE_DIRS}
)

target_link_libraries(
        ${PROJECT_NAME}_factor PUBLIC
        ${ROS_LIBRARIES}
        ctraj
        veta
        UFO::Map
        opengv
)

#####################
# libikalibr_viewer #
#####################
add_library(
        ${PROJECT_NAME}_viewer SHARED
        ${VIEWER_SRC_FILES}
)

target_include_directories(
        ${PROJECT_NAME}_viewer PUBLIC
        ${ROS_INCLUDE_DIRS}
        ${COMMON_INCLUDE_DIRS}
)

target_link_libraries(
        ${PROJECT_NAME}_viewer PUBLIC
        ${ROS_LIBRARIES}
        ctraj
        veta
        UFO::Map
        opengv
)

###################
# libikalibr_core #
###################
add_library(
        ${PROJECT_NAME}_core SHARED
        ${CORE_SRC_FILES}
)

target_include_directories(
        ${PROJECT_NAME}_core PUBLIC
        ${ROS_INCLUDE_DIRS}
        ${COMMON_INCLUDE_DIRS}
)

target_link_libraries(
        ${PROJECT_NAME}_core PUBLIC
        ${ROS_LIBRARIES}
        ctraj
        veta
        opengv
        UFO::Map
)

####################
# libikalibr_calib #
####################
add_library(
        ${PROJECT_NAME}_calib SHARED
        ${CALIB_SRC_FILES}
)

target_include_directories(
        ${PROJECT_NAME}_calib PUBLIC
        ${ROS_INCLUDE_DIRS}
        ${COMMON_INCLUDE_DIRS}
)

target_link_libraries(
        ${PROJECT_NAME}_calib PUBLIC
        ${ROS_LIBRARIES}
        ${PCL_LIBRARIES}
        ${OpenCV_LIBS}
        ctraj
        veta
        opengv
)

#####################
# libikalibr_solver #
#####################
add_library(
        ${PROJECT_NAME}_solver SHARED
        ${SOLVER_SRC_FILES}
)

target_include_directories(
        ${PROJECT_NAME}_solver PUBLIC
        ${ROS_INCLUDE_DIRS}
        ${COMMON_INCLUDE_DIRS}
)

target_link_libraries(
        ${PROJECT_NAME}_solver PUBLIC
        ${ROS_LIBRARIES}
        veta
        ctraj
        opengv
)

###################################
## catkin specific configuration ##
###################################
if (NOT IKALIBR_NO_ROS)
    catkin_package(
            CATKIN_DEPENDS roscpp rospy std_msgs geometry_msgs message_runtime
    )
endif ()

#######################
## Main Executable   ##
#######################
add_executable(
        ${PROJECT_NAME}_prog
        exe/solver/main.cpp
)

target_include_directories(
        ${PROJECT_NAME}_prog PUBLIC
        ${ROS_INCLUDE_DIRS}
        ${COMMON_INCLUDE_DIRS}
)

target_link_libraries(
        ${PROJECT_NAME}_prog PRIVATE
        ${PROJECT_NAME}_solver
        ${PROJECT_NAME}_calib
        ${PROJECT_NAME}_factor
        ${PROJECT_NAME}_core
        ${PROJECT_NAME}_viewer
        ${PROJECT_NAME}_sensor
        ${PROJECT_NAME}_config
        ${PROJECT_NAME}_util
        ${YAML_CPP_LIBRARIES}
)

##############################
## ROS-dependent executables #
##############################
# Modified: These tools require ROS and are only built when ROS is available
if (NOT IKALIBR_NO_ROS)
    add_executable(
            ${PROJECT_NAME}_learn
            exe/nofree/learn.cpp
            src/nofree/logo_svg.cpp
            src/nofree/data_collect_demo.cpp
    )
    add_executable(
            ${PROJECT_NAME}_lidar_map_viewer
            exe/tool/lidar_map_viewer.cpp
    )
    add_executable(
            ${PROJECT_NAME}_data_format_transformer
            exe/tool/data_format_transformer.cpp
    )
    add_executable(
            ${PROJECT_NAME}_bag_merge
            exe/tool/bag_merge.cpp
            src/nofree/bag_merge.cpp
    )
    add_executable(
            ${PROJECT_NAME}_bag_topic_downsample
            exe/tool/bag_topic_downsample.cpp
    )
    add_executable(
            ${PROJECT_NAME}_imgs_to_bag
            exe/tool/imgs_to_bag.cpp
    )
    add_executable(
            ${PROJECT_NAME}_imu_intri_calib
            exe/tool/imu_intri_calib.cpp
            src/nofree/imu_intri_calib.cpp
    )
    add_executable(
            ${PROJECT_NAME}_raw_inertial_to_bag
            exe/tool/raw_inertial_to_bag.cpp
    )

    # Link targets for ROS tools
    target_include_directories(${PROJECT_NAME}_learn PUBLIC ${catkin_INCLUDE_DIRS} ${CMAKE_CURRENT_SOURCE_DIR}/include)
    target_link_libraries(${PROJECT_NAME}_learn PRIVATE ${PROJECT_NAME}_util ${YAML_CPP_LIBRARIES})

    target_include_directories(${PROJECT_NAME}_lidar_map_viewer PUBLIC ${catkin_INCLUDE_DIRS} ${CMAKE_CURRENT_SOURCE_DIR}/include)
    target_link_libraries(${PROJECT_NAME}_lidar_map_viewer PRIVATE ${PROJECT_NAME}_util)

    target_include_directories(${PROJECT_NAME}_data_format_transformer PUBLIC ${catkin_INCLUDE_DIRS} ${CMAKE_CURRENT_SOURCE_DIR}/include)
    target_link_libraries(${PROJECT_NAME}_data_format_transformer PRIVATE
            ${PROJECT_NAME}_calib ${PROJECT_NAME}_factor ${PROJECT_NAME}_core
            ${PROJECT_NAME}_viewer ${PROJECT_NAME}_sensor ${PROJECT_NAME}_config
            ${PROJECT_NAME}_util ${YAML_CPP_LIBRARIES})

    target_include_directories(${PROJECT_NAME}_bag_merge PUBLIC ${catkin_INCLUDE_DIRS} ${CMAKE_CURRENT_SOURCE_DIR}/include)
    target_link_libraries(${PROJECT_NAME}_bag_merge PRIVATE ${PROJECT_NAME}_util ${YAML_CPP_LIBRARIES})

    target_include_directories(${PROJECT_NAME}_bag_topic_downsample PUBLIC ${catkin_INCLUDE_DIRS} ${CMAKE_CURRENT_SOURCE_DIR}/include)
    target_link_libraries(${PROJECT_NAME}_bag_topic_downsample PRIVATE ${PROJECT_NAME}_util ${YAML_CPP_LIBRARIES})

    target_include_directories(${PROJECT_NAME}_imgs_to_bag PUBLIC ${catkin_INCLUDE_DIRS} ${CMAKE_CURRENT_SOURCE_DIR}/include)
    target_link_libraries(${PROJECT_NAME}_imgs_to_bag PRIVATE ${PROJECT_NAME}_util)

    target_include_directories(${PROJECT_NAME}_imu_intri_calib PUBLIC ${catkin_INCLUDE_DIRS} ${CMAKE_CURRENT_SOURCE_DIR}/include)
    target_link_libraries(${PROJECT_NAME}_imu_intri_calib PRIVATE
            ${PROJECT_NAME}_calib ${PROJECT_NAME}_factor ${PROJECT_NAME}_core
            ${PROJECT_NAME}_viewer ${PROJECT_NAME}_sensor ${PROJECT_NAME}_config
            ${PROJECT_NAME}_util ${YAML_CPP_LIBRARIES})

    target_include_directories(${PROJECT_NAME}_raw_inertial_to_bag PUBLIC ${catkin_INCLUDE_DIRS} ${CMAKE_CURRENT_SOURCE_DIR}/include)
    target_link_libraries(${PROJECT_NAME}_raw_inertial_to_bag PRIVATE ${PROJECT_NAME}_util)
endif ()

#############
## Install ##
#############
install(TARGETS ${PROJECT_NAME}_prog
        RUNTIME DESTINATION bin
)

install(TARGETS
        ${PROJECT_NAME}_util
        ${PROJECT_NAME}_sensor
        ${PROJECT_NAME}_config
        ${PROJECT_NAME}_factor
        ${PROJECT_NAME}_viewer
        ${PROJECT_NAME}_core
        ${PROJECT_NAME}_calib
        ${PROJECT_NAME}_solver
        LIBRARY DESTINATION lib
        ARCHIVE DESTINATION lib
)

install(DIRECTORY include/
        DESTINATION include
        FILES_MATCHING PATTERN "*.h" PATTERN "*.hpp"
)

# Print build configuration
message(STATUS "===========================================")
message(STATUS "iKalibr Build Configuration:")
message(STATUS "  IKALIBR_NO_ROS: ${IKALIBR_NO_ROS}")
message(STATUS "  CMAKE_BUILD_TYPE: ${CMAKE_BUILD_TYPE}")
message(STATUS "  CMAKE_CXX_STANDARD: ${CMAKE_CXX_STANDARD}")
if (IKALIBR_NO_ROS)
    message(STATUS "  Input mode: Direct file input (PCD/Images/CSV)")
else ()
    message(STATUS "  Input mode: ROS bag + Direct file input")
endif ()
message(STATUS "===========================================")
