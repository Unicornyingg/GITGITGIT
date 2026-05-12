You can start a lot **before** the DJI M4/Matrice 4 + Manifold 3 connection works. Treat the DJI system as the final hardware target, but build your autonomy pipeline in simulation first.

Assuming you are on **Ubuntu 22.04**, I would build this stack:

```text
Ubuntu 22.04
→ ROS 2 Humble
→ Gazebo simulation
→ PX4 SITL drone model
→ RViz2 visualisation
→ RTAB-Map / SLAM pipeline
→ YOLO / object detection
→ Local planner / navigation logic
→ Later: replace simulator inputs with DJI PSDK inputs
```

PX4 officially recommends **ROS 2 Humble on Ubuntu 22.04** for PX4-ROS 2 development, and uses uXRCE-DDS for ROS 2 communication from PX4 v1.14 onward. ([PX4 Documentation][1])

## 1. What you should install first

### Core tools

```bash
sudo apt update
sudo apt upgrade -y

sudo apt install -y \
  git curl wget build-essential cmake python3-pip python3-venv \
  python3-colcon-common-extensions python3-rosdep python3-vcstool \
  terminator htop net-tools
```

### ROS 2 Humble

Use ROS 2 Humble if you are on Ubuntu 22.04. The official ROS docs list Ubuntu 22.04 as the target platform for Humble Debian packages. ([ROS Documentation][2])

Install the desktop version:

```bash
sudo apt install software-properties-common
sudo add-apt-repository universe

sudo apt update
sudo apt install -y ros-humble-desktop
```

Then add ROS to your shell:

```bash
echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

Set up `rosdep`:

```bash
sudo rosdep init
rosdep update
```

### Gazebo + ROS bridge

Gazebo is the simulator. ROS 2 docs include Gazebo tutorials for configuring robot simulations, and Gazebo’s own docs note that Gazebo Fortress is the safer pairing with ROS 2 Humble, while Harmonic can also be used with extra care. ([ROS Documentation][3])

For a practical ROS 2 Humble setup:

```bash
sudo apt install -y \
  ros-humble-gazebo-ros-pkgs \
  ros-humble-ros-gz \
  ros-humble-rviz2
```

### PX4 SITL

PX4 SITL gives you a simulated drone flight controller. You will not be simulating the exact DJI flight controller, but you can test autonomy logic, mapping, sensor pipelines, and waypoint control.

```bash
cd ~
git clone https://github.com/PX4/PX4-Autopilot.git --recursive
cd PX4-Autopilot

bash ./Tools/setup/ubuntu.sh
```

Restart your terminal after this.

Then test PX4 SITL:

```bash
cd ~/PX4-Autopilot
make px4_sitl gz_x500
```

If this launches a simulated drone in Gazebo, your flight simulation base is working.

## 2. Install mapping / SLAM tools

For real-time 3D mapping, start with **RTAB-Map** because it is one of the easier ROS-compatible choices for RGB-D / depth camera / stereo camera mapping. The `rtabmap_ros` package supports ROS 2, with Humble as a minimum target in its ROS 2 branch. ([GitHub][4])

Install:

```bash
sudo apt install -y \
  ros-humble-rtabmap-ros \
  ros-humble-image-transport \
  ros-humble-cv-bridge \
  ros-humble-pcl-ros \
  ros-humble-tf2-tools
```

RTAB-Map can generate 3D point clouds and occupancy maps for navigation use. ([ROS Wiki][5])

For now, your simulated drone should publish:

```text
/camera/color/image_raw
/camera/depth/image_raw
/camera/camera_info
/odom
/tf
```

Then RTAB-Map can consume those to build a map.

## 3. Install object/person detection

For a first prototype, use YOLO. Ultralytics provides a simple Python install route with:

```bash
pip install -U ultralytics
```

as the standard quickstart method. ([Ultralytics Docs][6])

Set it up in a virtual environment:

```bash
cd ~
python3 -m venv drone_ai_env
source ~/drone_ai_env/bin/activate

pip install --upgrade pip
pip install ultralytics opencv-python numpy
```

Test YOLO:

```bash
yolo predict model=yolov8n.pt source='https://ultralytics.com/images/bus.jpg'
```

For your project, keep the first detection goal simple:

```text
Input: RGB camera frame
Output: bounding boxes for person / object
Publish to ROS 2 topic: /detections
```

Later, combine detections with depth or map coordinates.

## 4. Important point about seeing through glass

Be careful with the phrase **“detect objects or people through windows or glass panes.”**

A normal RGB camera can sometimes see people through clear glass, depending on reflection, lighting, tint, dirt, and angle. But depth sensors and LiDAR often struggle with glass because transparent and reflective surfaces can cause missing, noisy, or misleading returns. Robotics literature specifically treats glass and transparent objects as a major perception challenge for navigation. ([arXiv][7])

Also, if you are using a thermal camera, normal window glass is usually a problem. Standard glass is largely opaque to long-wave infrared, so thermal cameras often see the glass surface or reflections rather than the person behind it. ([GSTiR][8])

So your simulation should test **two separate perception tasks**:

```text
Task A: Detect transparent/glass obstacle so the drone does not crash into it.

Task B: Detect visible people/objects behind glass using RGB vision, if the camera image actually shows them.
```

Do not rely only on depth or LiDAR for glass.

## 5. Install Navigation2, but use it carefully

Nav2 is mainly designed for 2D mobile robot navigation, not full 3D drone navigation. Still, it is useful for learning mapping + planning concepts. The Nav2 docs explain how to use SLAM with Nav2 to generate maps and navigate. ([Nav2 Documentation][9])

Install:

```bash
sudo apt install -y \
  ros-humble-navigation2 \
  ros-humble-nav2-bringup \
  ros-humble-slam-toolbox
```

For your drone, I would not make Nav2 the final navigation brain. Use it first to understand:

```text
map → localisation → path planning → obstacle avoidance
```

Then later move toward a 3D planner or a custom local planner that sends velocity/position setpoints.

## 6. Your simulation architecture

Build it like this:

```text
Gazebo world
  ↓
Simulated drone with RGB-D camera / LiDAR
  ↓
ROS 2 topics
  ↓
SLAM node
  ↓
3D map / occupancy map
  ↓
Object detection node
  ↓
Person/object detections
  ↓
Navigation planner
  ↓
PX4 offboard setpoints
```

Your ROS topics can look like this:

```text
/sim/camera/image_raw
/sim/camera/depth/image_raw
/sim/lidar/points
/sim/imu
/sim/odom
/map
/detections
/local_goal
/trajectory_setpoint
```

Later, when the DJI connection works, replace:

```text
Gazebo camera / simulated telemetry
```

with:

```text
DJI PSDK camera / telemetry / perception data
```

The Matrice 4 Series supports Payload SDK, Mobile SDK, and Cloud API, according to DJI’s support page. ([DJI Official][10]) Manifold 3 is also positioned by DJI as a compute module that supports media/sensor data acquisition, flight control, and route planning through PSDK-based applications. ([DJI Developer][11])

## 7. DJI side: what to prepare now

Even before the drone connects, you can prepare the DJI deployment layer.

Install/build the DJI Payload SDK on your Ubuntu machine:

```bash
cd ~
git clone https://github.com/dji-sdk/Payload-SDK.git
cd Payload-SDK
```

DJI’s Manifold 3 documentation says the Manifold development flow includes configuring the development environment, running sample programs, and packaging apps as DPK applications. ([DJI Developer][12]) DJI also documents a Manifold 3 quick demo for a full development flow. ([DJI Developer][13])

The final deployment flow will likely be:

```text
Build app
→ package as .dpk
→ install through DJI Pilot 2
→ run on Manifold 3
```

The Manifold 3 user manual says `.dpk` application packages can be copied to the remote controller or microSD card, then installed through **DJI Pilot 2 → Manifold 3 → Application Management**. ([DJI Download Center][14])

For later camera/perception integration, DJI’s PSDK docs include **Perception Data**, which covers grayscale images, LiDAR point clouds, and millimetre-wave radar data. ([DJI Developer][15])

## 8. What you can do immediately

Start with these milestones.

### Milestone 1: Basic drone simulation

Goal:

```text
Launch a drone in Gazebo and view it in RViz2.
```

Run:

```bash
cd ~/PX4-Autopilot
make px4_sitl gz_x500
```

Then open another terminal:

```bash
rviz2
```

### Milestone 2: Simulated sensor feed

Add or use a drone model with:

```text
RGB camera
depth camera
IMU
odometry
optional LiDAR
```

Check that topics exist:

```bash
ros2 topic list
```

Then inspect images:

```bash
rqt_image_view
```

Install if needed:

```bash
sudo apt install -y ros-humble-rqt-image-view
```

### Milestone 3: Run object detection on camera frames

Create a simple Python ROS 2 node:

```text
subscribe /camera/image_raw
run YOLO
publish /detections
```

For your first test, just draw bounding boxes and display the result.

### Milestone 4: Run RTAB-Map

Feed RGB + depth + camera info + odometry into RTAB-Map.

Expected output:

```text
3D point cloud
map
camera trajectory
loop closures if enabled
```

### Milestone 5: Add glass/window scenario

Create a Gazebo world with:

```text
room
glass pane
person model behind glass
solid wall
doorway
obstacles
```

Test separately:

```text
Can the detector see the person behind glass?
Does depth/LiDAR fail or become noisy at the glass?
Can the drone mark glass as an obstacle?
```

### Milestone 6: Build a hardware abstraction layer

This is very important. Do not write your whole project directly against DJI APIs.

Create an interface like:

```text
DroneInputProvider
  ├── GazeboInputProvider
  └── DjiPsdkInputProvider later
```

Your mapping and detection code should not care whether the image came from Gazebo or DJI.

Example structure:

```text
drone_project/
  src/
    perception/
      yolo_detector_node.py
      glass_detector_node.py
    mapping/
      rtabmap_launch.py
    navigation/
      local_planner_node.py
    interfaces/
      simulated_input_node.py
      dji_input_node_later.cpp
```

## 9. Recommended folder setup

```bash
mkdir -p ~/drone_ws/src
cd ~/drone_ws
colcon build
source install/setup.bash
echo "source ~/drone_ws/install/setup.bash" >> ~/.bashrc
```

Then create packages:

```bash
cd ~/drone_ws/src

ros2 pkg create drone_perception --build-type ament_python --dependencies rclpy sensor_msgs std_msgs cv_bridge

ros2 pkg create drone_navigation --build-type ament_python --dependencies rclpy geometry_msgs nav_msgs sensor_msgs

ros2 pkg create drone_sim_bringup --build-type ament_cmake
```

Build:

```bash
cd ~/drone_ws
colcon build
source install/setup.bash
```

## 10. What I would prioritise this week

Do not try to solve everything at once. Your first working demo should be:

```text
Ubuntu machine
→ Gazebo simulated drone
→ simulated RGB-D camera
→ RTAB-Map builds a 3D map
→ YOLO detects a person in the scene
→ RViz2 shows camera, map, and detections
```

Once that works, add:

```text
glass pane scenario
→ depth/LiDAR failure testing
→ RGB person detection through glass
→ glass obstacle segmentation
```

Then later, when Manifold connection works:

```text
replace Gazebo image topics with DJI PSDK camera/liveview topics
replace simulated odometry with DJI telemetry
package as .dpk
test on Manifold 3
```

The key is to make your autonomy code **sensor-source independent**. That way, your simulation work is not wasted when you move back to the DJI hardware.

[1]: https://docs.px4.io/main/en/ros2/user_guide "ROS 2 User Guide | PX4 Guide (main)"
[2]: https://docs.ros.org/en/humble/Installation/Ubuntu-Install-Debs.html?utm_source=chatgpt.com "Ubuntu (deb packages) — ROS 2 Documentation"
[3]: https://docs.ros.org/en/humble/Tutorials/Advanced/Simulators/Gazebo/Simulation-Gazebo.html?utm_source=chatgpt.com "Gazebo — ROS 2 Documentation: Humble documentation"
[4]: https://github.com/introlab/rtabmap_ros?utm_source=chatgpt.com "introlab/rtabmap_ros: RTAB-Map's ROS package."
[5]: https://wiki.ros.org/rtabmap_ros?utm_source=chatgpt.com "rtabmap_ros"
[6]: https://docs.ultralytics.com/quickstart/?utm_source=chatgpt.com "Install Ultralytics"
[7]: https://arxiv.org/html/2408.05608v1?utm_source=chatgpt.com "Real-time Transparent Obstacle Detection using Lidar ..."
[8]: https://www.gst-ir.net/news-events/infrared-knowledge/322.html?utm_source=chatgpt.com "Can thermal imaging cameras detect through glass?"
[9]: https://docs.nav2.org/tutorials/docs/navigation2_with_slam.html "Navigating while Mapping (SLAM) — Nav2 1.0.0 documentation"
[10]: https://www.dji.com/ca/support/product/matrice-4-series "Support for DJI Matrice 4 Series - DJI Australia"
[11]: https://developer.dji.com/doc/payload-sdk-tutorial/en/manifold-quick-start/manifold-product.html?utm_source=chatgpt.com "Manifold Product Overview - Payload SDK"
[12]: https://developer.dji.com/doc/payload-sdk-tutorial/en/manifold-quick-start/manifold-dpk.html?utm_source=chatgpt.com "Manifold Development Overview - Payload SDK"
[13]: https://developer.dji.com/doc/payload-sdk-tutorial/en/manifold-quick-start/quick-demo.html?utm_source=chatgpt.com "Quick Demo"
[14]: https://dl.djicdn.com/downloads/Manifold_3/20250828/Manifold_3_User_Manual_v1.2_en.pdf?utm_source=chatgpt.com "User Manual - DJI"
[15]: https://developer.dji.com/doc/payload-sdk-tutorial/en/advanced-function/perception-data.html?utm_source=chatgpt.com "Perception Data - Payload SDK"

