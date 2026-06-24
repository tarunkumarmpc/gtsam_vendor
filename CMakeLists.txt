cmake_minimum_required(VERSION 3.16)
project(gtsam_vendor)

find_package(ament_cmake REQUIRED)

# Force use of system Eigen (v3.4) over any local installs in /usr/local
# to maintain consistency with visual_graph_slam
if(EXISTS "/usr/share/eigen3/cmake/Eigen3Config.cmake")
  set(Eigen3_DIR "/usr/share/eigen3/cmake")
endif()

# Dependencies needed for GTSAM
find_package(TBB REQUIRED)
find_package(Eigen3 REQUIRED)
find_package(Boost REQUIRED COMPONENTS serialization) # GTSAM needs at least serialization

# Silence CMake developer and deprecation warnings from the subproject
set(CMAKE_WARN_DEPRECATED OFF CACHE BOOL "" FORCE)
set(CMAKE_SUPPRESS_DEVELOPER_WARNINGS ON CACHE BOOL "" FORCE)

# Silence compiler warnings from GTSAM source code (noisy in 4.2.0)
# Silence Boost deprecation notes (#pragma message)
add_compile_definitions(BOOST_ALLOW_DEPRECATED_HEADERS)

# Use generator expressions to apply C++ specific flags only to C++ files
add_compile_options(
  -Wno-deprecated-declarations
  -Wno-cpp
  -Wno-deprecated
  -Wno-array-bounds
  -Wno-stringop-overread
  -Wno-maybe-uninitialized
  -Wno-unused-variable
  $<$<COMPILE_LANGUAGE:CXX>:-Wno-overloaded-virtual>
  $<$<COMPILE_LANGUAGE:CXX>:-Wno-range-loop-construct>
)

# Path to the local GTSAM source code (now physically in the vendor directory)
set(GTSAM_SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/vendor/gtsam")

if(NOT EXISTS "${GTSAM_SOURCE_DIR}/CMakeLists.txt")
  message(FATAL_ERROR "GTSAM source code not found in ${GTSAM_SOURCE_DIR}. "
                      "Please ensure the source is extracted there.")
endif()

# Set GTSAM cache variables before add_subdirectory.
# We use FORCE to ensure GTSAM uses our strict configuration even if it tries 
# to define these itself via option() or set().
set(GTSAM_BUILD_TESTS OFF CACHE BOOL "Disable GTSAM tests" FORCE)
set(GTSAM_BUILD_EXAMPLES OFF CACHE BOOL "Disable GTSAM examples" FORCE)
set(GTSAM_BUILD_TIMING OFF CACHE BOOL "Disable GTSAM timing" FORCE)
set(GTSAM_BUILD_UNSTABLE OFF CACHE BOOL "Disable GTSAM unstable" FORCE)
set(GTSAM_INSTALL_MATLAB_TOOLBOX OFF CACHE BOOL "Disable Matlab toolbox" FORCE)
set(GTSAM_WITH_TBB ON CACHE BOOL "Enable TBB support" FORCE)
set(GTSAM_USE_SYSTEM_EIGEN ON CACHE BOOL "Use system Eigen" FORCE)
set(GTSAM_USE_SYSTEM_BOOST ON CACHE BOOL "Use system Boost" FORCE)
set(GTSAM_ALLOW_DEPRECATED_SINCE_V4 ON CACHE BOOL "Allow deprecated code since V4" FORCE)
set(GTSAM_USE_G2O ON CACHE BOOL "Enable g2o support in GTSAM" FORCE)

# Standard CMake build settings for vendor libraries
set(BUILD_SHARED_LIBS ON CACHE BOOL "Build GTSAM as shared libraries" FORCE)
set(CMAKE_POSITION_INDEPENDENT_CODE ON CACHE BOOL "Enable PIC" FORCE)

# Include GTSAM in the build. 
# This will also include GTSAM's install() rules, which will point to 
# the colcon install prefix (${CMAKE_INSTALL_PREFIX}).
add_subdirectory("${GTSAM_SOURCE_DIR}" "${CMAKE_CURRENT_BINARY_DIR}/gtsam-build")

# Export dependencies so downstream packages can find them
ament_export_dependencies(TBB Eigen3 Boost g2o_vendor)

# Create a hook to find GTSAM when this package is found
configure_file(
  cmake/gtsam_vendor-extras.cmake.in
  "${CMAKE_CURRENT_BINARY_DIR}/gtsam_vendor-extras.cmake"
  @ONLY
)

ament_package(
  CONFIG_EXTRAS "${CMAKE_CURRENT_BINARY_DIR}/gtsam_vendor-extras.cmake"
)
