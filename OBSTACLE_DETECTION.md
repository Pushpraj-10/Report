# TurtleBot3 Obstacle Detection and Maze Navigation Project Report

## 1. Project Introduction

This project was built around a simple robotics idea: make a TurtleBot3 robot understand the space around it and move safely without crashing into walls or objects. At the start, the robot was just a machine that could drive when told to drive. It did not “know” if there was something in front of it. We slowly taught it how to sense obstacles, decide whether the path was free, and turn away when something blocked its way.

We first built and tested everything in simulation using Gazebo. After that, we moved the same logic to the real TurtleBot3 hardware running on a Raspberry Pi. Along the way, we also explored a bigger goal: using localization and environment mapping to make the robot solve a maze instead of just avoiding obstacles. That part was the natural next step because once a robot can see where it is and understand the map, it can start planning routes through unknown spaces.

In short, this project moved through three stages:

1. Linux and ROS 2 setup on the Raspberry Pi
2. Obstacle detection in simulation and on the real TurtleBot3
3. Attempting maze solving using SLAM, localization, and navigation concepts

---

## 2. Starting from Linux and ROS 2 Setup

The work started on Ubuntu Linux and the Raspberry Pi inside the TurtleBot. The Raspberry Pi is the small computer that acts like the robot’s brain. We connected to it through SSH so that we could run commands remotely from the laptop. This made it easier to manage the robot without attaching a screen and keyboard directly to it.

Before doing anything else, we had to make the environment stable. We checked storage usage, cleaned unnecessary files, and removed old build folders that were taking up space. This mattered because the Pi’s storage can fill up quickly, and ROS 2 builds need a lot of room. If the system is too full, even simple robot programs can fail.

Next, we set up ROS 2 Jazzy on Ubuntu 24.04 and created a workspace for the project. ROS 2 is the framework that lets sensors, motors, and software nodes talk to each other. We sourced ROS in `.bashrc` so that every new terminal would know how to run `ros2` commands automatically. That way, the setup became repeatable and less painful.

We also installed `colcon`, the tool used to build ROS 2 packages. Once the workspace and environment were correct, we could start writing robot logic.

---

## 3. Obstacle Detection Logic

The first real robotics task was obstacle detection. The TurtleBot3 has a LiDAR sensor, which is like a rotating laser eye. It measures the distance to nearby objects all around the robot. LiDAR data comes in on the `/scan` topic.

Our custom node subscribed to `/scan` and divided the LiDAR view into three important parts:

- front
- left
- right

This gave the robot a very simple understanding of its surroundings. Instead of trying to think about the whole 360-degree scan at once, it only asked three questions:

- Is there something in front of me?
- Which side is more open?
- Which direction is safer to move?

If the front distance was smaller than a threshold, the robot stopped going straight and turned toward the side with more free space. If the path in front was clear, it continued forward. This is a classic reactive obstacle avoidance method, and it worked well for our initial project.

We also cleaned the sensor data by removing invalid values such as `inf` and `nan`. Real LiDAR sensors are not perfect, so this filtering step helped keep the robot stable.

---

## 4. Simulation Part in Gazebo

Simulation was a huge part of the project because it let us test everything without risking damage to the real robot. Gazebo is a robot simulator where TurtleBot3 can be placed in virtual environments. We used it to verify that the obstacle logic worked before moving to hardware.

At first, we tested in an empty world. This helped us confirm that the robot could spawn, the LiDAR topic existed, and the node could subscribe to `/scan`. In the empty world, the LiDAR returned mostly `inf`, which simply meant there were no nearby objects.

Then we tested with more interesting Gazebo worlds such as the TurtleBot3 house world. This was important because obstacle detection becomes more meaningful in a world with walls, corners, and narrow spaces. In those environments, the robot had to make real decisions about turning and escaping from blocked paths.

A lot of the simulation work was not just about the robot itself, but also about the environment around it. We had to deal with Gazebo resource paths, mesh files, and textures. Some models did not appear at first because the simulator could not find the correct files. We fixed this by setting up the correct environment variables and using the proper paths for TurtleBot3 models and Gazebo resources.

There were also WSL graphics problems. Gazebo GUI sometimes showed blank screens, missing entities, or missing textures because of rendering issues on Windows Subsystem for Linux. We learned that these GUI warnings often looked scary but were not always fatal. Sometimes the simulation backend was fine, and only the rendering was broken. We eventually handled this by cleaning up paths, using the right Gazebo resource configuration, and in some cases relying on RViz for visualization instead of depending only on the Gazebo GUI.

The simulation stage taught us a lot:
- how ROS 2 topics flow between nodes
- how sensors are represented in Gazebo
- how to debug missing models and textures
- how to separate a visual problem from a logic problem
- how to test robot behavior safely before hardware deployment

---

## 5. Moving from Simulation to the Real TurtleBot3

After the simulation version worked, we moved the same obstacle detection logic to the real TurtleBot3. This is where the project became much more meaningful because real robots are never as clean as simulations.

On the real robot, the TurtleBot3 bringup process started the LiDAR, motor drivers, odometry, and robot state publisher. We checked the `/scan` topic and confirmed that real LiDAR data was flowing. Once that worked, the obstacle node could run and make decisions based on actual sensor readings.

This stage revealed several real-world issues that simulation had hidden:
- message type mismatches
- QoS incompatibility
- topic conflicts on `/cmd_vel`
- LiDAR model and driver configuration problems
- graphics/path confusion between simulation and real hardware

At one point, the robot would print logs showing front, left, and right distances, but the wheels would not move. That taught us an important lesson: in robotics, seeing logs does not always mean the motors are receiving the right commands. We had to check whether the right topic type was being published, whether the controller was listening, and whether the robot hardware was actually ready to move.

Once the setup was correct, the real TurtleBot3 behaved exactly as expected:
- if an obstacle was close in front, it turned
- if one side had more free space, it preferred that direction
- if the path was clear, it moved ahead

That was a very satisfying milestone because the same logic worked both in simulation and on the actual robot.

---

## 6. Attempting Maze Solving with Localization and Mapping

After obstacle avoidance was working, we wanted to go one step beyond simple reactive behavior. Instead of just dodging obstacles, we wanted the robot to understand an environment like a maze, build a map, know where it was on that map, and then try to solve the maze.

This is where localization and environment mapping came in.

### Mapping
Mapping means the robot creates a representation of the world as it moves. In ROS, this is usually done through SLAM, which stands for Simultaneous Localization and Mapping. The robot uses its LiDAR and movement data to create a map of walls, openings, and corridors.

### Localization
Localization means the robot figures out where it is on the map. Once the map exists, the robot can estimate its position and orientation inside that space.

### Navigation
Navigation means planning how to move from one place to another while avoiding obstacles. This is typically handled by Nav2 in ROS 2.

We explored the idea of using a maze-like Gazebo world and then running SLAM to build the map while the robot explored it. The next step would have been to localize the robot within that map and then send it goals so it could reach an exit or a target area.

In simple terms, this was the difference between:
- “I see a wall, I turn away”
and
- “I know where I am in the maze, I know the layout, and I can plan a way out”

That is a much more advanced robotics problem, and it is exactly the kind of thing that makes a project impressive.

We also looked at TurtleBot3 prebuilt Gazebo worlds like the house world because they already contain corridor-like spaces and room layouts, which are a lot closer to a maze than an empty plane. These worlds were useful stepping stones for testing mapping and navigation behavior.

---

## 7. Challenges We Faced

A project like this always includes debugging. Some of the biggest issues were:
- ROS 2 environment not sourced correctly
- storage pressure on the Raspberry Pi
- Gazebo GUI instability on WSL
- model and texture path resolution problems
- LiDAR model mismatch
- QoS mismatch on `/scan`
- topic type conflicts on `/cmd_vel`

Each of these issues forced us to understand the system more deeply. That was actually a good thing because robotics is mostly about understanding how small pieces connect.

---

## 8. Final Outcome

By the end of the project, we had:
- a working obstacle detection node
- successful simulation testing in Gazebo
- successful deployment on the real TurtleBot3
- a strong understanding of how to move toward SLAM-based maze solving

The robot could detect obstacles with LiDAR and react intelligently by turning toward open space. We also laid the foundation for a much bigger goal: a maze-solving robot that uses localization and environment mapping rather than only simple avoidance.

---

## 9. Conclusion

This project started with basic Linux setup on a Raspberry Pi and ended with a real TurtleBot3 that could detect obstacles and move safely. Along the way, we learned simulation, ROS 2 communication, LiDAR processing, Gazebo model handling, real robot bringup, and the early ideas behind maze solving through mapping and localization.

The project was not just about making the robot move. It was about making the robot understand where it is, what is around it, and how to make safe decisions. That is the beginning of true autonomous robotics.
