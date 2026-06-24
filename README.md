# gtsam_vendor

`gtsam_vendor` is a ROS 2 `ament_cmake` vendor package that builds and exports the upstream **GTSAM** (Georgia Tech Smoothing and Mapping) library inside your workspace.

Downstream packages should use:

```cmake
find_package(gtsam_vendor REQUIRED)
```

without conflicting with system-wide installations.

## Features & Build Configuration

To minimize build times and focus purely on the core optimization functionality, this vendor package configures the GTSAM source build with specific, strict options:

- **Enabled**: `TBB` (multithreading), system `Eigen3`, system `Boost`, and `g2o` integration.
- **Disabled**: GTSAM Tests, Examples, Timing, Unstable modules, and the Matlab toolbox.
- **Position Independent Code (PIC)**: Enabled by default to safely build shared libraries.
- **Warning Suppression**: Silences verbose CMake and compiler warnings originating from the GTSAM source, keeping your ROS 2 workspace build logs clean and readable.

## Dependencies

Before building, ensure the following system dependencies are installed:

```bash
sudo apt update
sudo apt install -y libtbb-dev libboost-all-dev
```

*Note:* This package also depends on `g2o_vendor` from your workspace.

---

## Build

From workspace root:

```bash
cd /path/to/your_colcon_ws
colcon build --packages-select gtsam_vendor
```

## How to Use in a ROS 2 Package

Once `gtsam_vendor` is built and sourced, you can seamlessly use it in your downstream ROS 2 SLAM nodes (or any standard CMake project).

### 1. `package.xml`

Add the following to your downstream package's `package.xml`:

```xml
<depend>gtsam_vendor</depend>
```

### 2. `CMakeLists.txt`

In your downstream package's `CMakeLists.txt`, find the package and link against it. Note that you must enforce finding the workspace-built library to avoid conflicts with standard system installations in `/usr/` or `/usr/local/`.

Here is a robust example of how to link against the vendored GTSAM:

```cmake
find_package(ament_cmake REQUIRED)
find_package(gtsam_vendor REQUIRED)

# Force CMake to search for GTSAM in the workspace prefix
get_filename_component(_gtsam_vendor_prefix "${gtsam_vendor_DIR}/../../.." ABSOLUTE)
find_package(GTSAM CONFIG REQUIRED PATHS "${_gtsam_vendor_prefix}" NO_DEFAULT_PATH)

if(GTSAM_FOUND)
  message(STATUS "GTSAM found in workspace: ${GTSAM_DIR}")
  # Point directly to the exact library file to avoid target configuration mismatches
  set(GTSAM_LIBRARY_FILE "${_gtsam_vendor_prefix}/lib/libgtsam.so")
endif()

add_executable(my_slam_node src/my_slam_node.cpp)

target_include_directories(my_slam_node PUBLIC "${GTSAM_INCLUDE_DIR}")

target_link_libraries(my_slam_node
  "${GTSAM_LIBRARY_FILE}"
  TBB::tbb
)

ament_target_dependencies(my_slam_node rclcpp gtsam_vendor)

ament_package()
```

### Explanation of CMake Best Practices

- **`NO_DEFAULT_PATH`**: This is a critical argument. It forces CMake to ignore any `libgtsam-dev` packages installed globally on the OS and instead binds directly to the instance dynamically built inside the ROS 2 workspace.
- **`GTSAM_LIBRARY_FILE`**: We link directly to the `.so` file path. This is highly recommended to circumvent "imported target" pathing issues that can occasionally arise from GTSAM's complex internal CMake exports.

## Directory Structure

- `vendor/gtsam/`: Contains the actual C++ source code of GTSAM.
- `cmake/`: Contains the CMake extras hook for `ament` to correctly export the package downstream.
- `CMakeLists.txt`: Configures the vendor build process and applies compiler flag overrides.
- `package.xml`: Standard ROS 2 package definitions and dependencies.
