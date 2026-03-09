# Duckiebot ROS1 ↔ ROS2 Bridge Guide

This guide shows how to connect a **ROS2 Container** to a **ROS1 Duckiebot** using `ros1_bridge`.

At the end you will be able to:
- Receive Duckiebot ROS1 topics in ROS2
- Send ROS2 messages to ROS1 nodes

Architecture:
Duckiebot (ROS1 master)

│

│ 
ROS1

▼

ros1_bridge container 

│

│ ROS2

▼

Laptop ROS2 

>Important:  
**Do NOT run `roscore` manually for this setup.**  
If you run the container from within the Duckiebot the ROS1 master will remain on automatically.

## Download the ROS1 Bridge Image
Pull the official ROS bridge container:
```
docker pull ros:foxy-ros1-bridge
```
Verify:
```
docker images | grep ros1-bridge
```
## Define Robot Variables
Run this where you are running ROS2 (fill in the placeholders):
```
export HOSTNAME=<your_duckiebot_hostname>
export ROBOT_FQDN="${HOSTNAME}.local"

export ROBOT_IP=$(getent ahostsv4 ${ROBOT_FQDN} | awk 'NR==1{print $1}')
export LAPTOP_IP=$(ip route get ${ROBOT_IP} | awk '/src/ {print $7; exit}')

echo "Robot hostname: $ROBOT_FQDN"
echo "Robot IP: $ROBOT_IP"
echo "Laptop IP: $LAPTOP_IP"
```
Test connectivity:
```
ping -c 1 $ROBOT_FQDN
```
## Start Duckietown GUI Tools 
> Note: the `dts start_gui_tools` command starts a terminal (container) that is connected to the Duckiebot ROS network. In this terminal you can perform all ROS commands and have access to all ROS messages streaming on your Duckiebot. But if you are using a simulation run the relevant commands to start the process. Whether you use the physical Duckiebot or a simulation, you should end up with a container that we can communicate with. This guide here is specific for physical Duckeibots since it's the standard.

On the laptop:
```
dts start_gui_tools $HOSTNAME
```
Find the container:
```
docker ps
```
Enter it:
```
docker exec -it <gui_tools_container> bash
```
Inside the container:
```
source /opt/ros/noetic/setup.bash
rostopic list | head
```
You should see Duckiebot topics.

## Create a ROS1 Test Publisher
Inside the GUI tools container:
```
source /opt/ros/noetic/setup.bash

rostopic pub -r 2 /bridge_test std_msgs/String "data: hello_from_duckiebot"
```
## Start the ROS1 Bridge Container
On the host:
```
docker rm -f ros1_bridge 2>/dev/null || true

mkdir -p /tmp/ros1_bridge_home
mkdir -p /tmp/ros1_bridge_logs
```
Start the bridge:
```
docker run --rm -it \
  --network host \
  --ipc host \
  --user $(id -u):$(id -g) \
  --add-host ${ROBOT_FQDN}:${ROBOT_IP} \
  --name ros1_bridge \
  -e HOME=/tmp/ros1_bridge_home \
  -e ROS_HOME=/tmp/ros1_bridge_home/.ros \
  -e ROS_LOG_DIR=/tmp/ros1_bridge_logs \
  -e ROS_DOMAIN_ID=0 \
  -e RMW_IMPLEMENTATION=rmw_fastrtps_cpp \
  ros:foxy-ros1-bridge \
  bash
  ```
## Configure the Bridge
Inside the bridge container:
```
source /opt/ros/foxy/setup.bash
source /opt/ros/noetic/setup.bash

export ROS_MASTER_URI=http://$ROBOT_IP:11311
export ROS_IP=$LAPTOP_IP

export ROS_DOMAIN_ID=0
export RMW_IMPLEMENTATION=rmw_fastrtps_cpp

unset ROS_HOSTNAME
unset ROS_LOCALHOST_ONLY
```
Test ROS1 connectivity:
```
rostopic list | head
rostopic echo -n 1 /bridge_test
```
You should see:
```
data: hello_from_duckiebot
```
## Start the Dynamic Bridge
Still in the bridge container:
```
ros2 run ros1_bridge dynamic_bridge --bridge-all-topics
```
You should see:
```
created 1to2 bridge for topic '/bridge_test'
```
Leave this running.
## Test ROS1 → ROS2
Open a new host terminal:
```
source /opt/ros/humble/setup.bash

export ROS_DOMAIN_ID=0
export RMW_IMPLEMENTATION=rmw_fastrtps_cpp
unset ROS_LOCALHOST_ONLY
```
Stop the ROS2 CLI daemon:
```
ros2 daemon stop || true
pkill -f _ros2_daemon || true
```
Check topics:
```
ros2 topic list
```
You should see:
```
/bridge_test
```
Read the message:
```
ros2 topic echo /bridge_test std_msgs/msg/String --once
```
Expected output:
```
data: hello_from_duckiebot
```
## Test ROS2 → ROS1
Start a ROS1 listener (inside GUI tools container):
```
source /opt/ros/noetic/setup.bash
rostopic echo /bridge_back_test
```
On the host (ROS2):
```
source /opt/ros/humble/setup.bash

export ROS_DOMAIN_ID=0
export RMW_IMPLEMENTATION=rmw_fastrtps_cpp
unset ROS_LOCALHOST_ONLY

ros2 topic pub /bridge_back_test std_msgs/msg/String "{data: hello_from_ros2}"
```
Expected output in ROS1:
```
data: hello_from_ros2
```
#
After testing this successfully you can use this with real Duckeibot topics.