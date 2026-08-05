---
layout: doc
title: Scripts
description: >
  A beginner's introduction to Gazebo as a physics simulator before touching ROS2.
  You'll build a single wheel, configure it, and push it around **using only the GUI — zero code**.
hide_description: true
---

A beginner's introduction to Gazebo as a physics simulator before touching ROS2.
You'll build a single wheel, configure it, and push it around **using only the GUI — zero code**.

# Phase 0 — Drive a Simple Wheel in Gazebo (No Code)

A beginner's introduction to Gazebo as a physics simulator before touching ROS2.
You'll build a single wheel, configure it, and push it around **using only the GUI —
zero code**.

> This project has **no ROS2 involved**. It's pure Gazebo graphics + physics.
> Practice here, and later you'll recognize every knob in your TurtleBot3.

---

## 0. What you'll learn

- Gazebo is a **physics simulator**: you drop a body in, set mass/friction, and the
  physics engine (ODE) moves it.
- A model has a **visual** (what you see) and a **collision** (what physics uses).
  They must match, or the robot looks one way but behaves another.
- How to move objects with the GUI tools and with *Apply Force / Torque*.

---

## 1. Start a blank Gazebo

Open a terminal and run:

```bash
export GAZEBO_MODEL_PATH=/usr/share/gazebo-11/models
gazebo --verbose
```

An empty world opens: gray ground plane + sun.

**Camera controls (practice first):**
- Left-drag = rotate view
- Middle-drag / scroll = zoom
- Right-drag = pan

---

## 2. Create a cylinder (your "wheel")

- In the **toolbar above the scene**, click the **Cylinder** icon.
- Click on the ground to place it.
- The default is big and thick (1 m diameter × 1 m long) — we'll shape it.

---

## 3. Shape it into a wheel (Link Inspector)

1. Right-click the cylinder → **Open Link Inspector**.
2. In the **Collision** tab, expand `collision` → **Geometry** → **Cylinder**:
   - **Radius = 0.15**, **Length = 0.03**.
3. Do the **same** in the **Visual** tab:
   - **Radius = 0.15**, **Length = 0.03**.

> ⚠️ **Visual = what you see; Collision = what physics uses.** Update BOTH, or the
> wheel *looks* small but *behaves* big.

---

## 4. Make it roll like a wheel

A default cylinder stands upright (its axis points up). To lay it sideways so it rolls:

- Still in the inspector, under `collision` → **Pose** → set **Roll = 1.5707** (90°).
- Do the **same** for the **Visual pose** so what you see matches physics.

Now the disc stands upright on its rim, like a real wheel.

---

## 5. Configure physics (mass, friction, contacts)

In the **Link Inspector**:

- **Link tab → Inertial** → **Mass = 0.5 kg**. (Optional: adjust inertia; not required
  to make it roll.)
- **Link tab → Pose** → set **X=0, Y=0, Z=0.15** (so it rests on the ground, not sunk in).
- **Collision tab → Surface → Friction → ODE** → **mu = 1.0**, **mu2 = 1.0** (grip).
- **Collision tab → Surface → Contact → ODE**:
  - **kp = 100000**, **kd = 1.0**
  - **soft_cfm = 0**, **soft_erp = 0.2**
  - **max_vel = 1.0**, **min_depth = 0.001**

> The default contact stiffness the editor writes is `kp = 1e+13`, which makes the
> object lock/jitter instead of roll. The values above (from the known-good
> TurtleBot3 model) behave much better.

Click **OK** to confirm.

---

## 6. Move it WITHOUT coding (Apply Force / Torque)

1. Make sure the simulation is **not paused** (bottom **Play ▸** button).
2. Right-click the wheel → **Apply Force/Torque** → a dialog opens.
3. Under **Force**, type a large number (forces apply for a single tiny timestep,
   so values like **500–2000 N** are normal). E.g. `1500` in **X** → press
   **Apply Force** a few times → the wheel lurches and rolls forward.
4. Try **Torque** (e.g. `200` on Z) → the wheel spins.
5. Watch the **world tree** on the left for live pose/velocity.

---

## 7. Reset the wheel position (no files)

Once it rolls off screen, put it back:

**Quick — drag it:**
1. Select tool → click the wheel.
2. Press **`t`** (Translate) → drag the colored arrows back onto the ground.
3. Use **`r`** (Rotate) to straighten it if needed.

**Precise — set the pose:**
1. Right-click → **Open Link Inspector** → **Link tab → Pose**.
2. Enter **X=0, Y=0, Z=0.15** → **OK** → snaps back instantly.

**Frame it on screen:** right-click → **Move To** (or orbit with right-drag).

> 💡 **Pause** the sim while resetting, then **Play** when ready — otherwise it rolls
> away again while you work.

---

## 8. Experiment to reinforce learning

- Insert a **Box** (toolbar icon) and push the wheel into it to see collisions.
- Set friction `mu = 0.05` (ice) — the wheel slides/spins instead of rolling.
- Heavy wheel (mass 5 kg) vs. light wheel — notice how it responds to the same force.

---

## 9. How this maps to your TurtleBot3 (later)

Your Burger robot's wheels live in `models/turtlebot3_burger/model.sdf` — the same
cylinder + friction + mass recipe, plus a **plugin**
(`libgazebo_ros_diff_drive.so`) that subscribes to `/cmd_vel` and turns a
`Twist` message into wheel torques.

The **Apply Force** button here is the GUI/manual version of what that plugin does
automatically. So when you get to ROS2, every knob here will already feel familiar.