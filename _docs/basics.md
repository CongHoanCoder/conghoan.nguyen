---
layout: doc
title: Basics
description: >
  This chapter covers the basics of content creation with Hydejack.
video: https://www.youtube.com/watch?v=ueSmN_VxNPg&list=PLlqdnFs9xNwql5KET7v7zyl393y10qxtw&index=4
hide_description: true
---

This chapter covers the basics of content creation with Hydejack.


# ROS2 + Gazebo Robotics Learning Plan

A step-by-step plan covering simple control, LiDAR, localization, SLAM, and autonomous navigation.
**Estimated duration:** ~10–12 weeks at ~8–10 hours/week.

---

## Setup Recommendations (as of 2026)

| Component | Recommended | Alternative |
|-----------|-------------|-------------|
| ROS2 Distro | **Jazzy Jalisco** (LTS, until May 2029) | Humble (Ubuntu 22.04) |
| OS | Ubuntu 24.04 | Ubuntu 22.04 |
| Simulator | **Gazebo Harmonic** | Gazebo Harmonic |

> ⚠️ **Avoid "Gazebo Classic"** — it went end-of-life in January 2025. Make sure tutorials target the modern Gazebo (Harmonic/Sim).
> 💡 Install Ubuntu natively or dual-boot rather than WSL for smoother Gazebo graphics.

---

## Phase 1 — Foundations (Week 1–2)

**Goal:** Be comfortable with the ROS2 mental model and tooling before touching robots.

- Core concepts: nodes, topics, services, actions, parameters, the `ros2` CLI, pub/sub graph.
- Build system: workspaces, `colcon build`, packages, `ament`.
- Write nodes in **Python (rclpy)** first; revisit **C++ (rclcpp)** later.
- Debugging tools: `ros2 topic echo`, `ros2 node list`, `rqt_graph`, `ros2 bag`.

**Practice:** talker/listener pair, a node publishing to `/cmd_vel`, a service/client.

**Resource:** Official ROS2 Jazzy tutorials (docs.ros.org/en/jazzy) — Beginner CLI + Client Libraries.

---

## Phase 2 — Robot Description & Simulation (Week 3–4)

**Goal:** Model a robot and drop it into Gazebo.

- **URDF** → then **Xacro**: links, joints, inertials, collisions, visuals.
- Visualize in **RViz2**.
- **SDF** and Gazebo Harmonic worlds/models.
- **`ros_gz_bridge`** to connect ROS2 topics with Gazebo.

**Practice:** Build a differential-drive robot, add the diff-drive plugin, and drive it with `teleop_twist_keyboard`.

> Watch for version mismatches — use a supported pair (Jazzy+Harmonic or Humble+Harmonic).

---

## Phase 3 — Simple Control & Motion (Week 5)

**Goal:** Move the robot deliberately, not just by keyboard.

- **ros2_control**: hardware/sim interfaces, `diff_drive_controller`, `joint_state_broadcaster`.
- **TF2** transform tree: `map → odom → base_link → sensor frames`.
- TF is the backbone of localization, SLAM, and navigation — don't rush it.

**Practice:** Drive the robot in a square using odometry feedback; inspect TF with `ros2 run tf2_tools view_frames`.

---

## Phase 4 — Sensors & LiDAR (Week 6)

**Goal:** Give the robot perception.

- Add a simulated **2D LiDAR**; bridge `/scan` (LaserScan) into ROS2.
- Visualize the scan in RViz2.
- Add wheel odometry and optionally an IMU.
- Message types: `sensor_msgs/LaserScan`, `nav_msgs/Odometry`, `sensor_msgs/Imu`.

**Practice:** Simple obstacle-avoidance node that reads `/scan` and stops/turns near obstacles.

---

## Phase 5 — SLAM / Mapping (Week 7–8)

**Goal:** Build a map of an unknown environment.

- Understand what SLAM solves (estimate pose **and** build map simultaneously).
- Use **slam_toolbox** (standard ROS2 2D SLAM).
- Configure it with `/scan` + TF, drive around, watch the map build in RViz2, save the map.
- Compare online (real-time) vs offline (from a bag) mapping.

> Good SLAM depends on good odometry and a correct TF tree — Phase 3 issues surface here.

---

## Phase 6 — Localization (Week 9)

**Goal:** Locate the robot within a *known* map.

- Load the saved map.
- Use **AMCL** (`nav2_amcl`) — particle filter localization.
- Set an initial pose in RViz2; watch the particle cloud converge.
- Understand SLAM (unknown map) vs localization (known map).

---

## Phase 7 — Navigation with Nav2 (Week 10–11)

**Goal:** Full autonomous navigation — ties everything together.

- **Nav2 stack**: global/local costmaps, planners, controllers, behavior trees, recovery behaviors.
- Configure Nav2 with your robot, map, AMCL, and LiDAR.
- Send goal poses in RViz2; try waypoint following.
- Expect real parameter-tuning time — that's where deep understanding forms.

**Resource:** Nav2 docs (docs.nav2.org), including the "Navigating while Mapping" tutorial.

---

## Phase 8 — Capstone Project (Week 12+)

**Goal:** Consolidate everything into one integrated build.

- Autonomous exploration of an unknown environment (SLAM + Nav2), OR
- A "go fetch" waypoint mission, OR
- A multi-room patrol.
- Record with `ros2 bag`.
- If possible, port toward real hardware (TurtleBot-style or Raspberry Pi base) to tackle sim-to-real.

---

## Practical Tips

- **Commit to git often** — robotics stacks break in confusing ways; rollbacks are invaluable.
- **Master RViz2 + TF debugging early** — most navigation failures trace back to a broken TF tree or bad odometry.
- **Stick to one ROS2/Gazebo version pair** while learning to avoid dependency headaches.
- **Always verify** a tutorial targets the new Gazebo (Harmonic/Sim), not Gazebo Classic.

---

## Progress Checklist

- [ ] Phase 1 — Foundations
- [ ] Phase 2 — Robot Description & Simulation
- [ ] Phase 3 — Simple Control & Motion
- [ ] Phase 4 — Sensors & LiDAR
- [ ] Phase 5 — SLAM / Mapping
- [ ] Phase 6 — Localization
- [ ] Phase 7 — Navigation with Nav2
- [ ] Phase 8 — Capstone Project





# Phase 1 — Foundations (Detailed Plan)

**Goal:** Be comfortable with the ROS2 mental model and tooling before touching robots.

Target: **ROS2 Humble** on Ubuntu 22.04, **Python (rclpy)**, `colcon` build system.

## Target versions (verified on this machine)
| Component | Version here | Notes |
|---|---|---|
| ROS2 | **Humble** | `/opt/ros/humble`, sourced in `~/.bashrc` |
| Python | 3.10 | `rclpy` OK |
| Build tool | `colcon` | `/usr/bin/colcon` |
| Tooling | `ros2` CLI, `rqt_graph`, `ros2 bag`, `example_interfaces`, examples packages | all installed |

---

## Part 0 — Environment & CLI basics (½ day)

1. Source ROS2 in every terminal: `source /opt/ros/humble/setup.bash` (add to `~/.bashrc`).
2. Verify: `ros2 --version`, `python3 -c "import rclpy"`, `colcon` in PATH.
3. Create the workspace: `mkdir -p ~/Desktop/code/ros2_robotic/src`.
4. Sanity run of the built-in demo:
   - Terminal A: `ros2 run demo_nodes_py talker`
   - Terminal B: `ros2 run demo_nodes_py listener`
5. Learn the `ros2` command groups (`ros2 node --help`, `ros2 topic --help`, etc.) — this is your daily toolkit.

**Checkpoint:** two terminals exchange `/chatter`; you understand "node + topic" live.

## Part 1 — Core concepts: the ROS2 mental model (1 day)

Build one mental picture first; use the CLI to prove each piece.

- **Node** — a process doing one thing. `ros2 node list`, `ros2 node info /<node>`.
- **Topic** — publisher/subscriber bus, one-way, best-effort-ish data streaming. `ros2 topic list`, `ros2 topic info`, `ros2 topic echo`, `ros2 topic hz` (rate check), `ros2 topic pub` (inject test data).
- **Message** — the data type on a topic. `ros2 interface show std_msgs/msg/String`. Learn `std_msgs`, `geometry_msgs` (Twist, Pose, Point), `sensor_msgs` (LaserScan, Image), `nav_msgs` (Odometry).
- **Service** — request/reply, for short synchronous calls. `ros2 service list`, `ros2 service call`, `ros2 interface show example_interfaces/srv/AddTwoInts`.
- **Action** — long-running tasks with feedback + cancel (used later by Nav2). Just know it exists here; a full example in Part 4.
- **Parameter** — node configuration. `ros2 param list`, `ros2 param get/set`, `ros2 param dump`.
- **The graph** — everything above is the "ROS graph"; see it live with `rqt_graph` (start, then open the tab).
- **Namespaces & remapping** — `/robot1/cmd_vel` vs `/cmd_vel`; `--remap from:=to`. Don't skip; it powers multi-robot setups later.

**Checkpoint:** you can `echo` a live topic, call a service, set a parameter, and explain what `rqt_graph` draws.

## Part 2 — Workspaces, packages, `colcon` (½–1 day)

- **Workspace** (`src/`, `build/`, `install/`, `log/`). What each folder is.
- **Package anatomy:** `package.xml` (metadata + `<depend>`) and the build config.
- **Two build types:** `ament_python` (your primary — pure Python, simpler) and `ament_cmake` (used later for description packages with URDF/SDF). Know both.
- **Create one:**
  `ros2 pkg create my_first_pkg --build-type ament_python --dependencies rclpy std_msgs`
- **Build + source cycle:**
  - `colcon build --symlink-install` (symlink = edit Python without rebuilding — use it every time)
  - `source install/setup.bash`
  - `ros2 run my_first_pkg <node>`
- **Overlay vs underlay** — why you re-source after every build.

**Checkpoint:** an empty node you created runs with `ros2 run`.

## Part 3 — Write nodes in Python (rclpy) (1–2 days)

Start from the official tutorials (`docs.ros.org/en/humble` → Tutorials → Beginner: Client Libraries), then iterate.

1. **Talker/listener** — publisher + subscriber with `rclpy.init()`, `Node`, `create_publisher`, `create_subscription`, a timer callback, `rclpy.spin()`.
2. **Lifecycle discipline:** always `destroy_node()` / `rclpy.shutdown()`; don't spin in a loop.
3. **Custom messages in a `.msg` file** — create `my_interfaces` package, `ros2 interface show`, build, and use it.
4. **Service client/server** — `example_interfaces/srv/AddTwoInts`; `create_service` / `create_client` + `await` / `call`.
5. **Publisher with QoS:** default `depth=10` is fine now; later topics (e.g. `/scan`) need matching QoS — just notice the option exists.

Write it, break it, fix it. The goal is the *pattern*, not the perfect node.

**Checkpoint:** `rqt_graph` shows your nodes connected; you can `ros2 topic echo` your custom message.

## Part 4 — Service/client practice (½ day)

- Implement `AddTwoInts` server + client from Part 3.
- Call it cross-process: run the server node, call from `ros2 service call` AND from your client node.
- (Optional stretch) Minimal action: `ros2 run example_action_rclpy ...` to see feedback in action — skip if time-boxed.

## Part 5 — Debugging & logging toolkit (½–1 day)

- `ros2 node list / info`, `ros2 topic list -t / -v`, `ros2 topic echo --field`, `--qos-reliability` flag.
- `rqt_graph` — see the graph; fix "disconnected" nodes.
- `rqt_console` — node log output.
- `ROS2 log levels` — `rclpy.logging`, `--log-level`; use `get_logger().info/warn/error`.
- **`ros2 bag record/play`** — record your talker/`/cmd_vel` demo, play it back, `echo` the replayed topic. (This becomes your data-collection tool in SLAM phases.)
- `ros2 doctor` — health check when something's confusing.

**Checkpoint:** you can capture a bag of any running topic and replay it identically.

## Part 6 — Practice: `/cmd_vel` publisher (1 day — the milestone)

Your first "robot-facing" node, no robot yet.

1. Write a Python node that publishes `geometry_msgs/msg/Twist` to `/cmd_vel` on a timer (e.g. forward + rotate pattern).
2. Verify with `ros2 topic echo /cmd_vel` and `ros2 topic hz /cmd_vel`.
3. Add a parameter (`speed`, `direction`) and change it live with `ros2 param set` while the node runs.
4. Optionally subscribe to a `std_msgs/String` command topic and translate `"forward"`/`"stop"` into Twist — this foreshadows the teleop pattern used in Phase 2.
5. Record it all with `ros2 bag record /cmd_vel` and replay.

**Checkpoint:** you can steer a pretend robot from your own node, tune it via parameters at runtime, and record/replay the stream.

## Suggested package layout in `src/`
```
src/
  my_first_pkg/          # ament_python: talker/listener, cmd_vel publisher, param demo
  my_interfaces/         # ament_cmake: custom .msg / .srv definitions
```
`my_interfaces` uses `ament_cmake` (custom interfaces require it); your nodes live in the `ament_python` package.

## Troubleshooting cheat-sheet
- `Command 'ros2' not found` → didn't source `/opt/ros/humble/setup.bash` (or `install/setup.bash` for your packages).
- Node not listed → it crashed on startup; run it directly (`ros2 run ...` in foreground) to see the traceback.
- `rqt_graph` shows no nodes → ROS_DOMAIN_ID mismatch between terminals, or the graph is too fast to catch; use `ros2 topic list`.
- Topic echo empty → wrong QoS or the publisher only sent once before you subscribed; use `ros2 topic echo --qos-reliability reliable`.
- Rebuilt but `ros2 run` uses old code → `colcon build --symlink-install` + re-source `install/setup.bash`.
- Ports/RMW confusion → stick to default DDS until Phase 3+.

## Time budget
~2 weeks at 8–10 h/wk: Part 0–1 (1.5 d), Part 2 (1 d), Part 3 (2 d), Part 4 (0.5 d), Part 5 (1 d), Part 6 (1 d).

## Milestone checklist
- [ ] Workspace created, `colcon build --symlink-install` works
- [ ] `my_first_pkg` with a talker/listener pair runs
- [ ] Custom message interface built and used by your nodes
- [ ] Service client/server implemented and called cross-process
- [ ] `/cmd_vel` Twist publisher verified with `echo`, `hz`, `param set`, and bagged/replayed

## Resources
- Official: `docs.ros.org/en/humble` → Tutorials → Beginner CLI + Client Libraries (use Humble docs, not Jazzy/Foxy).
- `rclpy` API: docs.ros2.org/latest/api/rclpy.
- Book: "Programming Robots with ROS 2" (M. Quigley et al.) — beginner-friendly Humble-era text.
- If you need video: The Construct / Robotics Back-End (check the ROS2 Humble tag).

---

# Phase 2 — Robot Description & Simulation (Detailed Plan)

**Goal:** Model a robot and drop it into Gazebo.

Target: **ROS2 Humble** + **Ignition Gazebo Fortress** on Ubuntu 22.04, Python (rclpy).

## Target versions (verified on this machine)
| Component | Version here | Notes |
|---|---|---|
| ROS2 | **Humble** | `rclpy` OK, Python 3.10 |
| Simulator | **Ignition Gazebo Fortress 6.18** | runs via `ign gazebo` |
| Bridge | `ros_gz_bridge` 0.244 | Fortress build — uses `ignition.msgs.*` / `ign topic` |
| Tooling | `rviz2`, `xacro`, `robot_state_publisher`, `joint_state_publisher(_gui)`, `teleop_twist_keyboard`, `tf2_tools` | all installed |

> Skip Gazebo Classic 11 that's also on this box (EOL Jan 2025, uses URDF not SDF, and would bypass `ros_gz_bridge`). Stick to Fortress for the whole phase.

---

## Part 0 — Environment & workspace (½ day)

1. `source /opt/ros/humble/setup.bash` (add to `~/.bashrc`), verify `ros2 --version`, `ign gazebo --versions`.
2. Create the workspace: `ros2_robotic/src`, first `colcon build` run.
3. Create a description package (use `ament_cmake` so URDF/SDF get exported to `share/`, like `turtlebot3_description`):
   `ros2 pkg create my_diffbot_description --build-type ament_cmake`
4. Sanity check GUI works: `rviz2`, then `ign gazebo empty.sdf` (check `$DISPLAY`).

## Part 1 — URDF fundamentals (1–1.5 days)

Learn the mental model, then build.

- **Concepts:** a link has `visual`, `collision`, `inertial`; a joint joins parent→child with `origin` (xyz/rpy), `axis`, `type` (fixed, revolute, continuous, prismatic), `limit`.
- **Rules:** joints form a **tree** (one root, no cycles); URDF has no "world" — your root link is `base_link`.
- **Inertial gotchas (where beginners get burned in Gazebo):** inertia tensor must be positive-definite and non-zero; inertia origin must sit at the **center of mass**. Wrong values → robot falls over, sinks, or flings off.
- **Practice:** hand-write `urdf/robot.urdf`: `base_link` + `chassis` (box) + 2 wheels (cylinder, `continuous` joints, axis along Y) + caster spheres.
- **Verify:** `check_urdf robot.urdf`, `urdf_to_graphiz`, then a tiny launch with `robot_state_publisher` + `joint_state_publisher_gui` and view in RViz2.
- Resource: `docs.ros.org/en/humble` → Tutorials → URDF series.

## Part 2 — Xacro (½–1 day)

- Why: repetition + math. Learn `xacro:property`, `${...}` expressions, `xacro:macro`, `xacro:include`.
- Refactor Part 1: define `wheel_radius`, `wheel_base`, `track_width` as properties; macro for wheels.
- Validate: `xacro robot.xacro > robot.urdf` (diff with Part 1 file to confirm equality).

## Part 3 — RViz2 visualization (½ day)

- Add **RobotModel** display; set **Fixed Frame** to `base_link`; add **TF**, **Grid**.
- Drive joints with `joint_state_publisher_gui` sliders.
- Inspect the tree: `ros2 run tf2_tools view_frames`.
- Checkpoint: TF tree shows `base_link → chassis → left_wheel`, `right_wheel` (wheel joints are child frames only when joint states move).

## Part 4 — SDF & Gazebo Fortress (1–1.5 days)

- **SDF is Gazebo's native format**; `ign gazebo` loads SDF, not URDF. Learn world-file anatomy: `<world>` → `<scene>`, `<physics>`, `<gravity>`, lights, and models via `<include>` or inline `<model>`.
- Startup flags: `ign gazebo empty.sdf`, `-r` (run), `-s` (headless server), `-v` (verbosity for debugging).
- Two spawn approaches (learn the first, mention the second):
  - **Recommended:** put your robot inline in a **world file** and `ign gazebo -r my_world.sdf` — no services needed.
  - Create service: `ign service -s /world/<name>/create ...` with `ignition.msgs.EntityFactory`.
- **Version warning to internalize:** on Harmonic the plugin is `gz-sim-diff-drive-system` and tools are `gz sim`/`gz topic`; on Fortress it's `ignition-gazebo-*` and `ign topic`. This mismatch is the #1 source of copy-paste failures.
- Resource: gazebosim.org/docs/fortress (SDF + gz-sim tutorials).

## Part 5 — ros_gz_bridge (½–1 day)

- Why: ROS2 and Gazebo each have their own transport; the bridge relays topics with type translation.
- Learn the **YAML config**: `ros_topic_name`, `gz_topic_name`, `ros_type_name` (`geometry_msgs/msg/Twist`), `gz_type_name` (**`ignition.msgs.Twist`**), `direction`.
- Run it: `ros2 run ros_gz_bridge parameter_bridge --ros-args -p config_file:=config/bridge.yaml` (also try the CLI one-liner form).
- **Verify both directions:** `ign topic -l / -e` on the Gz side vs `ros2 topic list / echo` on the ROS side.
- Key type pairs to memorize: Twist↔Twist, Odometry↔Odometry, LaserScan↔LaserScan.

## Part 6 — Practice: differential-drive robot, drive it (2–3 days — the milestone)

1. In the SDF model, add the diff-drive plugin `ignition-gazebo-diff-drive-system`: `<left_joint>`, `<right_joint>`, `<wheel_separation>`, `<wheel_radius>`, `<topic>/cmd_vel</topic>`, `<odom_topic>/odom</odom_topic>`, `<odom_publish_frequency>`, plus TF publishing options.
2. Write `config/bridge.yaml` mapping:
   - `/cmd_vel` (ROS `Twist`) ↔ `/cmd_vel` (Gz `ignition.msgs.Twist`) — bidirectional
   - `/odom` (ROS `Odometry`) ↔ `/odom` (Gz `ignition.msgs.Odometry`) — gz→ros
3. Launch: `ign gazebo -r my_world.sdf` + bridge + `rviz2`.
4. Drive: `ros2 run teleop_twist_keyboard teleop_twist_keyboard`.
5. **Verify:** robot moves in the sim window; `ros2 topic echo /cmd_vel` and `/odom` update; RViz shows `odom → base_link` in TF.
6. Optional cleanups: a single `full.launch.py` that starts gz, bridge, rviz together; `colcon build` after each change.
- Resource: gazebosim.org/docs/fortress diff-drive example + ros_gz GitHub README (`gazebosim/ros_gz`).

## Suggested package layout in `src/`
```
my_diffbot_description/
  urdf/robot.urdf
  xacro/robot.xacro, macros.xacro
  sdf/robot.sdf, worlds/empty.sdf, worlds/robot_world.sdf
  config/bridge.yaml
  launch/display.launch.py, gazebo.launch.py, full.launch.py
  CMakeLists.txt, package.xml, (xacro files exported to share/)
```

## Troubleshooting cheat-sheet
- Robot sinks/flies → inertia missing or wrong, or COM offset.
- Wheels spin but robot stays still → diff-drive plugin not loaded (run `ign gazebo -v 3`, check plugin filename/params), or wheel joints not named correctly.
- No `/cmd_vel` on ROS side → bridge direction or message type mismatch.
- No `odom→base_link` TF → enable TF in the plugin or add a pose-publisher.
- Copy-paste tutorials behave oddly → you pasted `gz-sim-*`/`gz topic` (Harmonic) code; use `ignition-gazebo-*`/`ign topic`.

## Time budget
~2 weeks at 8–10 h/wk: Parts 0–1 (2 d), 2–3 (1.5 d), 4–5 (2 d), 6 (2–3 d).

## Milestone checklist
- [ ] Workspace + `my_diffbot_description` package builds
- [ ] `robot.urdf` validated and displays in RViz2
- [ ] Refactored `robot.xacro` produces identical URDF
- [ ] Robot drops into a Fortress world and rests on the ground
- [ ] Bridge forwards `/cmd_vel` and `/odom` both ways
- [ ] Diff-drive robot drives via `teleop_twist_keyboard`

## Resources
- URDF: `docs.ros.org/en/humble` → Tutorials → URDF series.
- SDF + gz-sim: gazebosim.org/docs/fortress.
- Diff-drive example: gazebosim.org/docs/fortress tutorial + `gazebosim/ros_gz` README.
- Articulated Robotics "Building a Robot with ROS2" series — great for URDF/Xacro intuition, but adapt it (targets newer ROS2 + Harmonic).
