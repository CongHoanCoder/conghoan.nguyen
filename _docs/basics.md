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
