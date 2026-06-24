# gtsam_vendor

ROS 2 `ament_cmake` vendor package for the [GTSAM](https://github.com/borglab/gtsam) (Georgia Tech Smoothing and Mapping) library. 

This package builds GTSAM from source and exports it as a local workspace target. This ensures deterministic builds, prevents ABI mismatches across downstream SLAM packages, and completely isolates the workspace from system-level `libgtsam-dev` installations.

---

## Build Configuration

To minimize compile times and strictly focus on the core optimization engine, the internal GTSAM build is configured with the following overrides:

- **Enabled**: `TBB` (multithreading), system `Eigen3`, system `Boost`
- **Disabled**: Tests, Examples, Timing, Unstable modules, Matlab toolbox
- **Compiler Flags**: `PIC` enabled by default; upstream CMake/compiler warnings suppressed to keep ROS 2 build logs clean.

---

## System Dependencies

Ensure the required system dependencies are installed before building the workspace:

```bash
sudo apt update
sudo apt install -y libtbb-dev libboost-all-dev git
```

---

## Build

Standard `colcon` build workflow:

```bash
cd /path/to/your_colcon_ws
colcon build --packages-select gtsam_vendor
source install/setup.bash
```

---

## Downstream Integration

Downstream packages must link against the vendored library to avoid runtime conflicts.

### `package.xml`

```xml
<depend>gtsam_vendor</depend>
```

### `CMakeLists.txt`

The exported CMake configuration enforces discovery within the workspace prefix. 

```cmake
find_package(ament_cmake REQUIRED)
find_package(gtsam_vendor REQUIRED)

# Force CMake to resolve GTSAM strictly from the workspace install prefix
get_filename_component(_gtsam_vendor_prefix "${gtsam_vendor_DIR}/../../.." ABSOLUTE)
find_package(GTSAM CONFIG REQUIRED PATHS "${_gtsam_vendor_prefix}" NO_DEFAULT_PATH)

if(NOT GTSAM_FOUND)
  message(FATAL_ERROR "GTSAM could not be found via gtsam_vendor.")
endif()

# Explicitly link against the generated shared object to circumvent imported target pathing issues
set(GTSAM_LIBRARY_FILE "${_gtsam_vendor_prefix}/lib/libgtsam.so")

add_executable(my_slam_node src/my_slam_node.cpp)

target_include_directories(my_slam_node PUBLIC "${GTSAM_INCLUDE_DIR}")
target_link_libraries(my_slam_node "${GTSAM_LIBRARY_FILE}" TBB::tbb)

ament_target_dependencies(my_slam_node rclcpp gtsam_vendor)
```

---

## Minimal Usage Example

A standard 2D factor graph initialization in a downstream node:

```cpp
#include <gtsam/geometry/Pose2.h>
#include <gtsam/nonlinear/NonlinearFactorGraph.h>
#include <gtsam/nonlinear/Values.h>
#include <gtsam/nonlinear/LevenbergMarquardtOptimizer.h>
#include <gtsam/slam/PriorFactor.h>

using namespace gtsam;

void optimize_graph() {
  NonlinearFactorGraph graph;
  
  // Apply a prior on the initial pose
  auto priorNoise = noiseModel::Diagonal::Sigmas(Vector3(0.3, 0.3, 0.1));
  graph.add(PriorFactor<Pose2>(1, Pose2(0, 0, 0), priorNoise));

  Values initialEstimate;
  initialEstimate.insert(1, Pose2(0.1, -0.1, 0.05));

  // Optimize
  LevenbergMarquardtOptimizer optimizer(graph, initialEstimate);
  Values result = optimizer.optimize();
}
```

---

## Maintenance 

- **Upstream Source**: The unmodified upstream source is maintained in `vendor/gtsam`. 
- **Updating GTSAM**: To upgrade the GTSAM version, update the `vendor/gtsam` tree, rebuild `gtsam_vendor`, and verify all downstream consumers.
