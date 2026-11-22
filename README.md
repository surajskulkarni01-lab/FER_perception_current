# fer_perception

ROS 2 Jazzy perception for the Franka Emika Robot (FER): a RealSense D405 on the hand,
object detection and a grasp pose network. It runs next to the
[fer_ros2](https://github.com/GKnerd/fer_ros2) platform, whose behavior trees use its
results.

Supported hosts: x86_64 PCs with an NVIDIA GPU, and the Jetson Orin on JetPack 7.2
(aarch64). Each architecture has its own Dockerfile (`docker/Dockerfile.x86_64`,
`docker/Dockerfile.aarch64`); `build_image.sh` picks the one matching `uname -m`.

## Pipeline and contract


The packages talk to fer_ros2 only through these two points:

| Interface | Kind | Name | Used in fer_ros2 by |
|---|---|---|---|
| `vision_msgs/Detection3DArray` | topic | `/perception/detections` | `fer_world_model`, which serves `DetectObjects` to the trees |
| `fer_interfaces/srv/GetGraspCandidates` | service | `/grasp/candidates` | the trees (served by `fer_grasp_planner` today) |

- Objects are boxes: a detection's pose is the centre of the object's bounding box, its size
  is the box size. Detections carry a class id and a score, no object id; the world model
  owns the ids.
- `GetGraspCandidates` gets an object id, looks the object up in the world model and returns
  candidates best first. It answers `NO_DATA` when there is no recent depth image.
- The full contract is in fer_ros2 `docs/Platform.md` §5 and in `fer_interfaces`, which the
  image builds from a fer_ros2 release.
- Between your own nodes, use standard types (`sensor_msgs`, `vision_msgs`,
  `geometry_msgs`). A new message type is agreed before it is written.

## Repository layout

```
docker/            Dockerfile.x86_64, Dockerfile.aarch64, build_image.sh, run_container.sh
env/               Fast DDS profiles
ros2_ws/src/       your ROS packages, one folder per package
ros2_ws/src/deps/  upstream code needed from source
data/              weights, datasets, recordings; not in git
log/               ROS logs of the container; not in git
```

`ros2_ws/build`, `install` and `log` are build output and not in git.

## Where things go

| What | Where |
|---|---|
| Your own ROS package | `ros2_ws/src/<package>` |
| apt package, or pip package with a rosdep key (`python3-…-pip`) | `package.xml` of the package that uses it; rosdep installs it when the image is built |
| pip package without a rosdep key | a pinned `pip install` line in both `docker/Dockerfile.*` files |
| Upstream code from source | `ros2_ws/src/deps/<name>` |
| Network weights, datasets, recordings | `data/` (in the container `~/data`) |

Upstream code in `deps/`:

- Copy it in without its `.git` folder, at a fixed commit or release. A folder with its own
  `.git` is not tracked by this repository.
- Keep its license file, and add a row to `NOTICE.md`: name, license, URL, commit. Check the
  license before copying; several grasp network repositories allow research use only.
- Do not change the code in `deps/`. A needed change goes into your own package, or is noted
  in `NOTICE.md` with what was changed and why.
- colcon builds every package under `ros2_ws/src`, `deps/` included.

## Host setup

On each host, once:

- Docker and the NVIDIA Container Toolkit, with the `nvidia` runtime registered.
- The RealSense udev rules, installed with the camera unplugged:

  ```bash
  sudo mkdir -p /etc/apt/keyrings
  curl -sSf https://librealsense.realsenseai.com/Debian/librealsense.pgp | sudo tee /etc/apt/keyrings/librealsense.pgp > /dev/null
  echo "deb [signed-by=/etc/apt/keyrings/librealsense.pgp] https://librealsense.realsenseai.com/Debian/apt-repo $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/librealsense.list
  sudo apt-get update && sudo apt-get install librealsense2-udev-rules
  ```

- x86_64: NVIDIA driver 580 or newer (CUDA 13).
- Jetson Orin: JetPack 7.2 (Jetson Linux 39.2, Ubuntu 24.04, CUDA 13.2). JetPack 7.0 and 7.1
  do not support Orin; on JetPack 6 the image runs without GPU.

Connect the D405 to a USB 3 port.

## Build and run

```bash
git clone <repository URL> fer_perception && cd fer_perception
./docker/build_image.sh
./docker/run_container.sh
```

The scripts must be run from inside the `fer_perception` folder.

- The image is `fer_perception/ros:jazzy_<arch>` (`x86_64` or `aarch64`), the container
  `fer_perception`.
- The image contains ROS 2 Jazzy, RViz, the RealSense driver (`realsense2_camera`,
  `realsense2_description`), PyTorch and torchvision with CUDA 13 (cu130 on x86_64, cu132 on the Orin, whose wheels support sm_87), and `fer_interfaces` in
  the underlay `~/deps_ws`. There is no CUDA compiler in the image. Versions are the `ARG`
  lines in `docker/Dockerfile.<arch>`.
- `build_image.sh` builds `ros2_ws` in the image and copies `build/`, `install/` and `log/`
  back to the host `ros2_ws/`. Files you change during an image build are overwritten with
  the state from the start of the build.
- `run_container.sh` mounts `ros2_ws`, `env`, `data`, `log` and `/dev`, and gives the
  container the GPU and the display. The container uses the host network, so it sees the
  fer_ros2 nodes on the same `ROS_DOMAIN_ID`.

A second terminal in the running container:

```bash
docker exec -it fer_perception bash
```

Rebuild and test in the container, before every pull request:

```bash
colcon build --symlink-install --merge-install
colcon test --merge-install && colcon test-result --verbose
```

Rebuild the image when `docker/Dockerfile.<arch>` or a `package.xml` dependency changes.

## Camera and RViz

```bash
ros2 launch realsense2_camera rs_launch.py camera_name:=d405 pointcloud.enable:=true
rviz2
```

In RViz: Fixed Frame `d405_link`, an Image display on `/camera/d405/color/image_raw`, a
PointCloud2 display on `/camera/d405/depth/color/points`. Depth is
`/camera/d405/depth/image_rect_raw`. Depth, color and pointcloud run at 30 Hz.

## Logs

- ROS logs (nodes, launch) go to `log/` in the repository: `run_container.sh` mounts it at
  `~/logs` and sets `ROS_LOG_DIR` to it.
- colcon logs go to `ros2_ws/log`.

## Check on a new host

```bash
rs-enumerate-devices --compact
python3 -c "import torch; print(torch.cuda.is_available(), torch.cuda.get_arch_list()); a = torch.randn(1024, 1024, device='cuda'); print((a @ a).sum())"
```

The first lists the D405 with its firmware. The second prints `True` and a finite number
when PyTorch runs on the GPU.

## License

Apache-2.0, see `LICENSE`. Third-party notices: `NOTICE.md`.
