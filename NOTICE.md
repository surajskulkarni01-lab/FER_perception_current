# Notices

RealSense D405 perception stack for the FER. Copyright 2026 Georgios Katranis.
Licensed under the Apache License 2.0 (`LICENSE`), except where a package states otherwise.

## Derived work in this repository

- `docker/`: derived from internal tooling of Proximity Robotics & Automation GmbH
  (Copyright 2025), Apache-2.0, by way of fer_ros2. Each file carries its modification notice.

## External components

Not part of this repository; `docker/Dockerfile.<arch>` downloads them when the image is built.

| Component | License | Source |
|---|---|---|
| ROS 2 Jazzy (`ros:jazzy-ros-base`), RViz | Apache-2.0, BSD-3-Clause | https://github.com/ros2 |
| librealsense2 | Apache-2.0 | https://github.com/realsenseai/librealsense |
| realsense-ros (`realsense2_camera`, `realsense2_description`) | Apache-2.0 | https://github.com/realsenseai/realsense-ros |
| vision_msgs | Apache-2.0 | https://github.com/ros-perception/vision_msgs |
| fer_interfaces (fer_ros2) | Apache-2.0 | https://github.com/GKnerd/fer_ros2 |
| PyTorch | BSD-3-Clause | https://github.com/pytorch/pytorch |
| torchvision | BSD-3-Clause | https://github.com/pytorch/vision |
| NVIDIA CUDA runtime libraries (dependencies of the PyTorch wheels) | NVIDIA Software License Agreement | https://pypi.org/user/nvidia/ |
