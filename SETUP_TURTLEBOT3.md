# TurtleBot3 Obstacle Detection: Full Setup and Run Guide

This guide takes you from a fresh SBC OS flash to a successful runtime of:
1. TurtleBot3 base bringup
2. This repository's Rust obstacle-detection/navigation app
3. Optional Python obstacle node workflow

Primary target in this document:
- Ubuntu 22.04 (64-bit)
- ROS2 Humble
- TurtleBot3 Burger

## Required Execution Order (Always Follow)

Every run must follow this order:
1. Power ON robot
2. SSH into robot
3. Start bringup: `ros2 launch turtlebot3_bringup robot.launch.py`
4. Verify topics/sensors
5. Run your node (Rust or Python)

If step 3 is skipped, movement and sensors will not work.

---

## PHASE 0: One-Time Setup (Do This Once)

### 0.1 Flash Ubuntu on SBC

1. Flash Ubuntu 22.04 (64-bit) to your SBC (Raspberry Pi / Jetson) SD card.
2. Boot the SBC.
3. Complete initial OS setup (username, password, network).
4. Open a terminal on the robot and update packages:

```bash
sudo apt update
sudo apt upgrade -y
```

### 0.2 Install ROS2 Humble + TurtleBot3 Packages

Install ROS2 desktop and TurtleBot3 packages:

```bash
sudo apt update
sudo apt install -y ros-humble-desktop
sudo apt install -y ros-humble-turtlebot3-msgs ros-humble-turtlebot3-bringup ros-humble-turtlebot3-teleop
```

If your package mirror supports wildcard patterns, this also works:

```bash
sudo apt install -y ros-humble-turtlebot3*
```

Add ROS and robot model to shell startup:

```bash
echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
echo "export TURTLEBOT3_MODEL=burger" >> ~/.bashrc
source ~/.bashrc
```

### 0.3 Set Device Permissions (Important)

```bash
sudo usermod -a -G dialout $USER
```

Then reboot once:

```bash
sudo reboot
```

### 0.4 Connect Hardware

Confirm all hardware is connected before runtime:
1. USB: SBC <-> OpenCR
2. LiDAR cable connected
3. Motor cables connected
4. Battery connected and charged

### 0.5 Clone This Project

On your development machine and/or robot:

```bash
git clone https://github.com/Pushpraj-10/turtlebot3-obstacle-detection.git
cd turtlebot3-obstacle-detection
```

---

## PHASE 0B: Build and Deploy This Repository (One-Time or On Updates)

This project executable name is:
- `ros2_cmd_vel_publisher`

Choose one build method.

### Option A: Direct Build on Robot (Recommended for Humble)

Use this when building directly on the SBC running ROS2 Humble.

1. Install Rust:

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source "$HOME/.cargo/env"
```

2. Install basic native build dependencies:

```bash
sudo apt update
sudo apt install -y build-essential pkg-config libclang-dev cmake ffmpeg libv4l-dev
```

3. Build release binary:

```bash
cd ~/turtlebot3-obstacle-detection
source /opt/ros/humble/setup.bash
cargo build --release
```

4. Binary output:

```text
~/turtlebot3-obstacle-detection/target/release/ros2_cmd_vel_publisher
```

### Option B: Cross-Compile on PC, Copy Binary to Robot

Use repository scripts for containerized build flow.

1. Build container image and start container:

```bash
cd ~/turtlebot3-obstacle-detection
./build-image.sh
```

2. Sync local source into container:

```bash
./docker-compile.sh
```

3. Build inside container:

```bash
./build-project.sh
```

4. Copy build artifact out of container:

```bash
./copy-result.sh
```

Important note:
- `copy-result.sh` currently contains lab-specific SCP targets/IPs.
- Edit it for your own robot IP/user or use manual copy:

```bash
container_id=$(docker ps -q --filter "ancestor=my-system")
mkdir -p ./output-binary
docker cp $container_id:/rust-example/target/aarch64-unknown-linux-gnu/release/ros2_cmd_vel_publisher ./output-binary/
scp ./output-binary/ros2_cmd_vel_publisher ubuntu@<robot_ip>:~/turtlebot3-obstacle-detection/output-binary/
```

Compatibility note:
- Current Docker tooling in this repository is based on Ubuntu 24.04 + ROS2 Jazzy.
- Your runtime target here is Ubuntu 22.04 + ROS2 Humble.
- For a strict Humble-only environment, Option A is usually safer.

### 0.6 Runtime Data/Model Path Requirement (Do Not Skip)

The app loads config from `../data/config.json` and model from config path `../data/best_model.onnx`.

To run successfully without code changes:
1. Keep `data/` directory next to `src/` (as in this repository layout).
2. Start the binary with current directory set to `src/`.

Example:

```bash
cd ~/turtlebot3-obstacle-detection/src
../target/release/ros2_cmd_vel_publisher
```

If you use cross-compiled output:

```bash
cd ~/turtlebot3-obstacle-detection/src
../output-binary/ros2_cmd_vel_publisher
```

### 0.7 MongoDB Logging (Optional for First Run)

Operationally, Mongo logging is optional, but current code initializes Mongo at startup.

If you want zero-friction first run:
1. Ensure outbound internet is available on robot
2. Use a valid Mongo URI in the code

If you want offline first run:
1. Temporarily disable Mongo initialization/logging lines in `src/main.rs`
2. Build again and run local-only navigation pipeline

---

## PHASE 1: Starting the Robot (Every Time)

### Step 1: Power ON

1. Turn on robot battery.
2. Wait 30-60 seconds.

### Step 2: Find Robot IP

On robot terminal:

```bash
hostname -I
```

### Step 3: SSH from PC

```bash
ssh ubuntu@<robot_ip>
```

### Step 4: Source ROS (if needed)

If `.bashrc` already has ROS source line, this is optional. Otherwise:

```bash
source /opt/ros/humble/setup.bash
export TURTLEBOT3_MODEL=burger
```

### Step 5: Start Mandatory Bringup

In Terminal 1 (keep running):

```bash
ros2 launch turtlebot3_bringup robot.launch.py
```

This starts core robot stack:
1. Motor driver
2. LiDAR node
3. Odometry
4. TF

---

## PHASE 2: Verify System Health (Do Not Skip)

In Terminal 2:

### 2.1 Check Topics

```bash
ros2 topic list
```

You should see at least:
1. `/scan`
2. `/cmd_vel`
3. `/odom`
4. `/tf`

### 2.2 Check LiDAR Stream

```bash
ros2 topic echo /scan
```

If ranges are updating, LiDAR is healthy.

### 2.3 Quick Movement Test (Teleop)

In another terminal (Terminal 3):

```bash
ros2 run turtlebot3_teleop teleop_keyboard
```

Keys:
1. `w` forward
2. `a` left
3. `d` right

If robot responds correctly, base stack is ready for custom nodes.

---

## PHASE 3: Run Your Programs

### 3A. Run This Repository (Rust First)

Prerequisite:
1. Bringup terminal from PHASE 1 is still running.

Run command (choose binary location):

```bash
# Direct build binary
cd ~/turtlebot3-obstacle-detection/src
../target/release/ros2_cmd_vel_publisher
```

or

```bash
# Cross-compiled binary copied to output-binary/
cd ~/turtlebot3-obstacle-detection/src
../output-binary/ros2_cmd_vel_publisher
```

What this Rust app uses:
1. LiDAR subscription: `/scan`
2. Odom subscription: `/odom`
3. Velocity publishing: `/cmd_vel`
4. Camera + YOLO detection pipeline

### 3B. Optional: Python Obstacle Node Workflow

Use this only if you also want a pure Python ROS node path.

1. Create workspace (one-time):

```bash
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws
colcon build
echo "source ~/ros2_ws/install/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

2. Create package:

```bash
cd ~/ros2_ws/src
ros2 pkg create my_robot --build-type ament_python
```

3. Create node file: `~/ros2_ws/src/my_robot/my_robot/obstacle_node.py`

```python
import rclpy
from rclpy.node import Node
from sensor_msgs.msg import LaserScan
from geometry_msgs.msg import Twist


class ObstacleAvoider(Node):
    def __init__(self):
        super().__init__('obstacle_avoider')

        self.sub = self.create_subscription(
            LaserScan, '/scan', self.callback, 10
        )

        self.pub = self.create_publisher(Twist, '/cmd_vel', 10)

    def callback(self, msg):
        twist = Twist()

        front = min(msg.ranges[0:20] + msg.ranges[-20:])

        if front < 0.5:
            twist.angular.z = 0.5
        else:
            twist.linear.x = 0.2

        self.pub.publish(twist)


def main():
    rclpy.init()
    node = ObstacleAvoider()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()
```

4. Register entry point in `setup.py`:

```python
entry_points={
    'console_scripts': [
        'obstacle_node = my_robot.obstacle_node:main',
    ],
},
```

5. Build and source:

```bash
cd ~/ros2_ws
colcon build
source install/setup.bash
```

6. Run node:

```bash
ros2 run my_robot obstacle_node
```

---

## Debug Checklist (When Things Fail)

Robot not moving:

```bash
ros2 topic echo /cmd_vel
```

No LiDAR data:

```bash
ros2 topic echo /scan
```

No OpenCR communication:

```bash
ls /dev/ttyACM*
```

Sensor frequency health:

```bash
ros2 topic hz /scan
```

Check odom updates:

```bash
ros2 topic echo /odom
```

---

## Clean Industry-Style Terminal Workflow

Use three terminals every session:
1. Terminal 1: bringup (`ros2 launch turtlebot3_bringup robot.launch.py`)
2. Terminal 2: your main node (Rust binary or Python node)
3. Terminal 3: diagnostics (`ros2 topic echo`, `ros2 topic hz`, serial checks)

This keeps operations stable, debuggable, and production-like.

---

## Final Boot-to-Run Checklist

1. Robot flashed with Ubuntu 22.04 and ROS2 Humble installed
2. `dialout` permission configured and reboot completed
3. Hardware fully connected
4. Bringup running without errors
5. `/scan`, `/cmd_vel`, `/odom`, `/tf` visible
6. Teleop movement test passed
7. Rust app launched from `src/` working directory
8. (Optional) Python obstacle node runs correctly

If all 8 checks pass, setup is complete and your robot stack is operational.
