Yes, you can run **Ubuntu 20.04 + ROS 2** using Docker on your Ubuntu 26.04 machine.

For Ubuntu 20.04, the ROS 2 version is **ROS 2 Foxy**. Just note that Foxy has reached end-of-life, so it is fine for compatibility/testing, but not ideal for a long-term production system. ([ROS Documentation][1])

## 1. Install Docker on your Ubuntu 26.04 host

Run this on your normal Ubuntu 26.04 terminal:

```bash
sudo apt update
sudo apt install -y docker.io python3-pip pipx x11-xserver-utils
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
```

Then log out and log back in.

Test Docker:

```bash
docker run hello-world
```

## 2. Install Rocker for GUI support

Rocker helps Docker containers use GUI apps like **RViz2** and **Gazebo**. It is commonly used for ROS Docker workflows with X11, NVIDIA, user permissions, and mounted folders. ([GitHub][2])

```bash
pipx ensurepath
pipx install rocker
```

Close and reopen your terminal, then check:

```bash
rocker --help
```

## 3. Pull the ROS 2 Foxy Docker image

Use the OSRF ROS image:

```bash
docker pull osrf/ros:foxy-desktop
```

The `foxy-desktop` image is based on ROS 2 Foxy desktop packages for Ubuntu 20.04/Focal. ([Docker Hub][3])

## 4. Create a persistent workspace folder

This folder stays on your Ubuntu 26.04 machine, but will be mounted inside the Ubuntu 20.04 ROS 2 container.

```bash
mkdir -p ~/drone_foxy_ws/src
```

## 5. Run the Ubuntu 20.04 + ROS 2 Foxy container

```bash
rocker --x11 --user --network=host --privileged \
  --volume ~/drone_foxy_ws:/home/$USER/drone_foxy_ws \
  osrf/ros:foxy-desktop
```

You are now inside a Docker container running ROS 2 Foxy.

Check:

```bash
lsb_release -a
```

You should see Ubuntu 20.04.

Check ROS 2:

```bash
echo $ROS_DISTRO
```

Expected:

```text
foxy
```

## 6. Test ROS 2 inside the container

In the container, run:

```bash
ros2 run demo_nodes_cpp talker
```

Open another terminal on your host and enter the same container again:

```bash
rocker --x11 --user --network=host --privileged \
  --volume ~/drone_foxy_ws:/home/$USER/drone_foxy_ws \
  osrf/ros:foxy-desktop
```

Then run:

```bash
ros2 run demo_nodes_py listener
```

If the listener receives messages, ROS 2 is working.

## 7. Test RViz2

Inside the container:

```bash
rviz2
```

If RViz2 opens, your GUI setup is working.

If it fails with a display error, run this on your host first:

```bash
xhost +local:docker
```

Then try launching the container again.

## 8. Set up your ROS 2 workspace

Inside the container:

```bash
cd ~/drone_foxy_ws
colcon build
source install/setup.bash
```

Add it to `.bashrc` inside the container:

```bash
echo "source /opt/ros/foxy/setup.bash" >> ~/.bashrc
echo "source ~/drone_foxy_ws/install/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

## 9. Create your first ROS 2 package

Inside the container:

```bash
cd ~/drone_foxy_ws/src

ros2 pkg create drone_perception \
  --build-type ament_python \
  --dependencies rclpy sensor_msgs std_msgs geometry_msgs cv_bridge
```

Then build:

```bash
cd ~/drone_foxy_ws
colcon build
source install/setup.bash
```

## 10. What to install next inside the container

For your drone simulation work, install these inside the container:

```bash
sudo apt update
sudo apt install -y \
  python3-colcon-common-extensions \
  python3-rosdep \
  ros-foxy-rviz2 \
  ros-foxy-image-transport \
  ros-foxy-cv-bridge \
  ros-foxy-pcl-ros \
  ros-foxy-tf2-tools
```

For Python vision tools:

```bash
python3 -m pip install --upgrade pip
python3 -m pip install opencv-python numpy ultralytics
```

## Important recommendation

Since you are using ROS 2, the setup is:

```text
Ubuntu 26.04 host
→ Docker + Rocker
→ Ubuntu 20.04 container
→ ROS 2 Foxy
→ RViz2 / Gazebo / mapping / detection code
```

For your project, start by getting these working in order:

```text
1. ROS 2 talker/listener
2. RViz2 opens
3. Camera topic simulation
4. YOLO detection node
5. Mapping/SLAM node
6. Drone simulation later
```

The command you will use most often is:

```bash
rocker --x11 --user --network=host --privileged \
  --volume ~/drone_foxy_ws:/home/$USER/drone_foxy_ws \
  osrf/ros:foxy-desktop
```

[1]: https://docs.ros.org/en/foxy/Installation.html?utm_source=chatgpt.com "Installation — ROS 2 Documentation: Foxy documentation"
[2]: https://github.com/osrf/rocker?utm_source=chatgpt.com "osrf/rocker: A tool to run docker containers with overlays ..."
[3]: https://hub.docker.com/r/osrf/ros/?utm_source=chatgpt.com "osrf/ros - Docker Image"
