Option B: Use Docker, recommended for simulation/dev

Docker lets you run Ubuntu 20.04 inside your Ubuntu 26.04 system without replacing your OS. Docker’s official Ubuntu guide recommends installing Docker Engine through its apt repository.

1. Install Docker on Ubuntu 26.04

Run this on your host machine:

sudo apt update
sudo apt install -y ca-certificates curl gnupg

Add Docker’s repository:

sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

sudo chmod a+r /etc/apt/keyrings/docker.gpg

Add the Docker source:

echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

Install Docker:

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

Allow your user to run Docker:

sudo usermod -aG docker $USER

Then log out and log back in.

Test:

docker run hello-world
2. Pull a Ubuntu 20.04 ROS Noetic image

For ROS 1 Noetic:

docker pull osrf/ros:noetic-desktop-full

Then run it:

docker run -it --name ros-noetic-sim osrf/ros:noetic-desktop-full bash

Inside the container, check:

lsb_release -a

You should see Ubuntu 20.04.

Check ROS:

rosversion -d

Expected:

noetic
3. Run GUI apps like RViz/Gazebo

Normal Docker containers do not automatically show GUI windows. For robotics, use rocker, which is made to run ROS Docker containers with GUI support. Rocker supports X11, NVIDIA, CUDA, user mapping, home-directory mounting, and network options.

Install rocker:

sudo apt update
sudo apt install -y python3-pip
pip3 install rocker

Then run ROS Noetic with GUI support:

rocker --x11 --user --network=host --privileged osrf/ros:noetic-desktop-full

Inside the container, test RViz:

rviz

If RViz opens, GUI forwarding works.

If you have an NVIDIA GPU, use:

rocker --x11 --nvidia --user --network=host --privileged osrf/ros:noetic-desktop-full
4. Make your workspace persistent

By default, files inside a Docker container can be lost if you delete the container. Create a workspace on your host:

mkdir -p ~/drone20_ws

Run the container with that folder mounted:

rocker --x11 --user --network=host --privileged \
  --volume ~/drone20_ws:/home/$USER/drone20_ws \
  osrf/ros:noetic-desktop-full

Inside the container:

cd ~/drone20_ws
mkdir -p catkin_ws/src
cd catkin_ws
catkin_make

Source it:

echo "source /opt/ros/noetic/setup.bash" >> ~/.bashrc
echo "source ~/drone20_ws/catkin_ws/devel/setup.bash" >> ~/.bashrc
source ~/.bashrc
5. For PX4 simulation on Ubuntu 20.04

Inside your Ubuntu 20.04 container or VM:

cd ~
git clone https://github.com/PX4/PX4-Autopilot.git --recursive
cd PX4-Autopilot
bash ./Tools/setup/ubuntu.sh

Then restart the container or source your shell again.

For older PX4/Gazebo Classic workflows on 20.04, PX4’s docs note that the setup script installs Gazebo Classic 11 on Ubuntu 20.04.

Try:

make px4_sitl gazebo

or depending on PX4 version:

make px4_sitl_default gazebo
My recommendation for you

Use this setup:

Ubuntu 26.04 host
→ Docker + rocker
→ Ubuntu 20.04 ROS Noetic container
→ Gazebo/RViz/PX4 simulation
→ Mount ~/drone20_ws so your files persist

The fastest starting command after Docker is installed:

mkdir -p ~/drone20_ws

rocker --x11 --user --network=host --privileged \
  --volume ~/drone20_ws:/home/$USER/drone20_ws \
  osrf/ros:noetic-desktop-full

Then inside the container:

roscore

Open another rocker terminal and test:

rviz

That gives you a working Ubuntu 20.04 ROS Noetic simulation environment without deleting Ubuntu 26.04.
