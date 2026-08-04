---
layout: doc
title: Basics
description: >
  This chapter covers the basics of content creation with Hydejack.
video: https://www.youtube.com/watch?v=ueSmN_VxNPg&list=PLlqdnFs9xNwql5KET7v7zyl393y10qxtw&index=4
hide_description: true
---

This chapter covers the basics of content creation with Hydejack.

# TurtleBot3 Beginner Robotics Plan (ROS2 + Gazebo)

A gentle, results-first path into robotics using the pre-built TurtleBot3 robot.
**Estimated duration:** ~8–10 weeks at ~6–8 hours/week.

---

## Setup Recommendations (as of 2026)

| Component | Recommended | Notes |
|-----------|-------------|-------|
| ROS2 Distro | **Humble** (easiest for beginners) | Most TurtleBot3 tutorials target Humble; Ubuntu 22.04 |
| ROS2 Distro (alt) | **Jazzy** | Now supports TB3 + Gazebo Harmonic; use the `jazzy` branch of the TB3 repos |
| Simulator | **Gazebo Harmonic** | Avoid Gazebo Classic (EOL Jan 2025) |
| Robot model | `burger` (simplest) | Set with `export TURTLEBOT3_MODEL=burger` |

> 💡 **Beginner tip:** If you want the smoothest ride with the most tutorials that "just work," start with **Humble**. Move to Jazzy once you're comfortable.
> 📖 Primary reference: the ROBOTIS TurtleBot3 e-Manual (now on ROBOTIS Docs) and the Nav2 docs.

---

## Phase 0 — Install & First Launch (Week 1)

**Goal:** Get the environment installed and see TurtleBot3 appear in Gazebo.

- Install Ubuntu (native or dual-boot preferred over WSL for graphics).
- Install ROS2 (Humble or Jazzy) following the official docs.
- Install TurtleBot3 packages: `turtlebot3`, `turtlebot3_msgs`, `turtlebot3_simulations`.
- Set `export TURTLEBOT3_MODEL=burger` in your `~/.bashrc`.
- Launch an empty/simple TurtleBot3 world in Gazebo.

**Milestone:** You see the TurtleBot3 robot sitting in a Gazebo world. 🎉

---

## Phase 1 — Drive It Around (Week 1–2)

**Goal:** Control the robot and understand what's moving under the hood.

- Launch the TurtleBot3 world.
- Run `teleop_twist_keyboard` (or the TB3 teleop node) and drive it with the keyboard.
- Open **RViz2** and watch the robot, its LaserScan (`/scan`), and TF frames update live.
- Peek at the data: `ros2 topic list`, `ros2 topic echo /cmd_vel`, `ros2 topic echo /scan`.

**Milestone:** You can drive the robot and see its sensor data streaming. This is your "the whole thing works" moment.

---

## Phase 2 — Run SLAM (Week 2–3)

**Goal:** Build a map of the simulated world — using ready-made launch files.

- Launch the TurtleBot3 SLAM demo (uses **slam_toolbox** under the hood).
- Drive the robot slowly around the whole world using teleop.
- Watch the map build up in RViz2 in real time.
- **Save the map** to disk (`map.pgm` + `map.yaml`).

**Milestone:** You have your own saved map file. Don't worry yet about *how* SLAM works — just experience it.

---

## Phase 3 — Autonomous Navigation with Nav2 (Week 3–4)

**Goal:** Make the robot drive itself to a goal.

- Launch the TurtleBot3 Nav2 demo with your saved map.
- In RViz2: set the **initial pose** (2D Pose Estimate), then set a **goal** (Nav2 Goal).
- Watch the robot plan a path and drive there while avoiding obstacles.
- Try **waypoint following** (multiple goals in sequence).

**Milestone:** The robot navigates autonomously. You've now seen the *entire* pipeline: drive → map → localize → navigate. 🚀

---

## ⏸️ Checkpoint: Now You Understand the Big Picture

You've made everything work using pre-built tools. From here, we slow down and learn **why** each piece works, so you can eventually build your own.

---

## Phase 4 — ROS2 Fundamentals (Week 4–5)

**Goal:** Understand the tools you've been using.

- Concepts: nodes, topics, services, actions, parameters.
- The `ros2` CLI and `rqt_graph` to see the node/topic graph of the running TB3 demos.
- Workspaces, packages, and `colcon build`.
- Write your **first Python node**: subscribe to `/scan` and print the closest obstacle distance.

**Practice:** Write a simple obstacle-avoidance node — drive forward, stop/turn when `/scan` shows something close.

---

## Phase 5 — TF & Robot Structure (Week 5–6)

**Goal:** Understand how the robot knows where it is.

- The **TF2** transform tree: `map → odom → base_link → sensor frames`.
- Inspect the TB3 TF tree: `ros2 run tf2_tools view_frames`.
- Look inside the TurtleBot3 **URDF/Xacro** to see how links, joints, and the LiDAR are defined.
- Understand odometry (`/odom`) and how it drifts over time.

**Why this matters:** TF is the backbone of SLAM, localization, and navigation. Most beginner bugs are TF/odometry problems.

---

## Phase 6 — Understand SLAM & Localization (Week 6–7)

**Goal:** Learn what was happening in Phases 2–3.

- **SLAM (slam_toolbox):** how it estimates pose *and* builds the map from `/scan` + odometry.
- **Localization (AMCL / `nav2_amcl`):** particle-filter localization on a *known* map.
- Reload your saved map, run AMCL, set the initial pose, and watch the particle cloud converge as you drive.
- Understand the key distinction: **SLAM = unknown map**, **AMCL = known map**.

---

## Phase 7 — Understand & Tune Nav2 (Week 7–8)

**Goal:** Look inside the navigation stack you ran in Phase 3.

- Nav2 building blocks: global/local **costmaps**, **planners**, **controllers**, **behavior trees**, recovery behaviors.
- Open the TB3 Nav2 config/param files and change values (e.g. inflation radius, max speed) — observe the effect.
- Learn "Navigating while Mapping" (SLAM + Nav2 together).

**Practice:** Tune the params so the robot navigates more smoothly or hugs walls less. Tuning is where real understanding forms.

---

## Phase 8 — Make It Your Own (Week 8+)

**Goal:** Go beyond the demos.

- Create a **custom Gazebo world** and map it with SLAM.
- Write a node that sends navigation goals **programmatically** (via the Nav2 action API) instead of clicking in RViz2.
- Build a small mission: "patrol these 3 waypoints" or "explore until the map is complete."
- (Stretch goal) Try a **real TurtleBot3** — the Nav2 docs have a "Navigating with a Physical TurtleBot3" tutorial.

---

## Practical Tips

- **Always `export TURTLEBOT3_MODEL=burger`** in every new terminal (put it in `~/.bashrc`).
- **Source your workspace** (`source install/setup.bash`) in every terminal — a top beginner gotcha.
- **One terminal per launch file** — you'll juggle several (Gazebo, SLAM/Nav2, teleop, RViz2).
- **Commit to git often**; robotics stacks break in confusing ways.
- **Match versions strictly:** Humble+Harmonic or Jazzy+Harmonic. On Jazzy, use the `jazzy` branch of the TurtleBot3 repos.
- **Drive slowly during SLAM** — fast motion produces messy maps.

---

## Progress Checklist

- [ ] Phase 0 — Install & First Launch
- [ ] Phase 1 — Drive It Around
- [ ] Phase 2 — Run SLAM
- [ ] Phase 3 — Autonomous Navigation with Nav2
- [ ] Phase 4 — ROS2 Fundamentals
- [ ] Phase 5 — TF & Robot Structure
- [ ] Phase 6 — Understand SLAM & Localization
- [ ] Phase 7 — Understand & Tune Nav2
- [ ] Phase 8 — Make It Your Own
