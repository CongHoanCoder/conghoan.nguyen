---
layout: doc
title: Scripts
description: >
  Goal: drive a single wheel in Gazebo by typing `Force X` and `Torque` values in a
  terminal and pressing **Enter**. The wheel moves/rotates via physics.
hide_description: true
---


# Phase 1 — Control a Simple Wheel in Gazebo (ROS2 Pub/Sub)

Goal: drive a single wheel in Gazebo by typing `Force X` and `Torque` values in a
terminal and pressing **Enter**. The wheel moves/rotates via physics.

You build it from scratch: a **wheel model**, a **Python package** with a
**publisher** and a **subscriber**, then wire them to Gazebo through a ROS2 service.

```
force_cmd (publisher)  --/wheel_cmd-->  apply_wrench (subscriber)  --service-->  Gazebo physics
     |  reads Force X / Torque              calls /apply_link_wrench              pushes the wheel
```

---

## 0. Environment (verify once)

- ROS2 **Humble** (Gazebo Classic 11) on Ubuntu 22.04.
- `ROS_DISTRO` should read `humble`. It is set automatically by
  `source /opt/ros/humble/setup.bash` — **do not set it manually**.

```bash
echo $ROS_DISTRO                 # expect: humble
source /opt/ros/humble/setup.bash
source ~/Desktop/code/ros2_robotic/turtlebot3_ws/install/setup.bash
```

`.bashrc` already sources ROS + the workspace, so a new terminal is ready to go.

---

## 1. The wheel model (`models/unit_cylinder/`)

Location: `~/Desktop/code/ros2_robotic/models/unit_cylinder/`

A Gazebo model is a folder with two files:

```
models/unit_cylinder/
├── model.config   # tells Gazebo the model name + which SDF to load
└── model.sdf      # the model itself: link, mass, friction, visual, collision
```

### `model.config`

```xml
<?xml version="1.0" ?>
<model>
  <name>unit_cylinder</name>
  <version>1.0</version>
  <sdf version="1.7">model.sdf</sdf>
  <author><name></name><email></email></author>
  <description></description>
</model>
```

### `model.sdf`

Key idea: a model has one **link** (a rigid body). The link has a **visual**
(what you see) and a **collision** (what physics uses) — they must match, or the
wheel looks one way but behaves another.

```xml
<?xml version='1.0'?>
<sdf version='1.7'>
  <model name='unit_cylinder'>
    <link name='link'>
      <inertial>
        <mass>0.5</mass>
        <inertia>
          <ixx>0.00285</ixx> <ixy>0</ixy> <ixz>0</ixz>
          <iyy>0.00285</iyy> <iyz>0</iyz>
          <izz>0.005625</izz>
        </inertia>
        <pose>0 0 0 0 0 0</pose>
      </inertial>
      <self_collide>0</self_collide>
      <enable_wind>0</enable_wind>
      <kinematic>0</kinematic>
      <pose>0 0 0.15 0 0 0</pose>          <!-- disc center 0.15 m above ground -->
      <gravity>1</gravity>
      <visual name='visual'>
        <geometry>
          <cylinder><radius>0.15</radius><length>0.03</length></cylinder>
        </geometry>
        <material>
          <script><name>Gazebo/Grey</name><uri>file://media/materials/scripts/gazebo.material</uri></script>
        </material>
        <pose>0 0 0 1.5707 0 0</pose>       <!-- lie the disc on its side -->
      </visual>
      <collision name='collision'>
        <laser_retro>0</laser_retro>
        <max_contacts>10</max_contacts>
        <pose>0 0 0 1.5707 0 0</pose>       <!-- collision must match visual -->
        <geometry>
          <cylinder><radius>0.15</radius><length>0.03</length></cylinder>
        </geometry>
        <surface>
          <friction>
            <ode><mu>1.0</mu><mu2>1.0</mu2><fdir1>0 0 0</fdir1><slip1>0</slip1><slip2>0</slip2></ode>
          </friction>
          <bounce><restitution_coefficient>0</restitution_coefficient><threshold>1e+06</threshold></bounce>
          <contact>
            <ode>
              <soft_cfm>0</soft_cfm><soft_erp>0.2</soft_erp>
              <kp>100000</kp><kd>1.0</kd><max_vel>1.0</max_vel><min_depth>0.001</min_depth>
            </ode>
            <bullet>
              <split_impulse>1</split_impulse><split_impulse_penetration_threshold>-0.01</split_impulse_penetration_threshold>
              <soft_cfm>0</soft_cfm><soft_erp>0.2</soft_erp><kp>100000</kp><kd>1.0</kd>
            </bullet>
          </contact>
        </surface>
      </collision>
    </link>
    <static>0</static>
    <allow_auto_disable>0</allow_auto_disable>
  </model>
</sdf>
```

> **Why it was fixed:** the original GUI-saved file had a collision body that did
> not match the visual (wrong radius, wrong orientation, buried in the ground, and
> a huge contact stiffness `kp=1e+13`). This version keeps collision == visual,
> rests on the ground (`pose z=0.15` = radius), and uses sane contact values.

To be found by Gazebo, the model's **parent folder** must be on the model path:

```bash
export GAZEBO_MODEL_PATH=/usr/share/gazebo-11/models:$HOME/Desktop/code/ros2_robotic/models
```

(This line is already in `~/.bashrc`.)

---

## 2. The Python package (`src/wheel_control/`)

Location: `~/Desktop/code/ros2_robotic/turtlebot3_ws/src/wheel_control/`

```
wheel_control/
├── package.xml
├── setup.py
├── setup.cfg
├── resource/
│   └── wheel_control
└── wheel_control/
    ├── __init__.py
    ├── force_cmd.py      # publisher: reads Force X / Torque from keyboard
    └── apply_wrench.py   # subscriber: calls /apply_link_wrench
```

### `package.xml`

```xml
<?xml version="1.0"?>
<?xml-model href="http://download.ros.org/schema/package_format3.xsd" schematypens="http://www.w3.org/2001/XMLSchema"?>
<package format="3">
  <name>wheel_control</name>
  <version>0.0.1</version>
  <description>Publisher/subscriber example to push a wheel with force and torque in Gazebo.</description>
  <maintainer email="cong@todo.todo">cong</maintainer>
  <license>Apache-2.0</license>

  <exec_depend>rclpy</exec_depend>
  <exec_depend>geometry_msgs</exec_depend>
  <exec_depend>gazebo_msgs</exec_depend>

  <export><build_type>ament_python</build_type></export>
</package>
```

### `setup.py`

```python
import os
from glob import glob
from setuptools import setup

package_name = 'wheel_control'

setup(
    name=package_name,
    version='0.0.1',
    packages=[package_name],
    data_files=[
        ('share/ament_index/resource_index/packages', ['resource/' + package_name]),
        ('share/' + package_name, ['package.xml']),
        (os.path.join('share', package_name), glob('launch/*.launch.py')),
    ],
    install_requires=['setuptools'],
    zip_safe=True,
    maintainer='cong',
    maintainer_email='cong@todo.todo',
    description='Publisher/subscriber example to push a wheel with force and torque in Gazebo.',
    license='Apache-2.0',
    entry_points={
        'console_scripts': [
            'force_cmd = wheel_control.force_cmd:main',
            'apply_wrench = wheel_control.apply_wrench:main',
        ],
    },
)
```

### `setup.cfg`

```ini
[develop]
script_dir=$base/lib/wheel_control
[install]
install_scripts=$base/lib/wheel_control
```

### `resource/wheel_control` and `wheel_control/__init__.py`

Both are empty files (just present, as required by `ament_python`).

### `wheel_control/force_cmd.py` — the publisher

```python
import rclpy
from rclpy.node import Node
from geometry_msgs.msg import Wrench


class ForceCmd(Node):
    """Reads force X and torque Z from the keyboard and publishes them on /wheel_cmd."""

    def __init__(self):
        super().__init__('force_cmd')
        self.pub = self.create_publisher(Wrench, '/wheel_cmd', 10)

    def run(self):
        print('Make sure Gazebo is running first: ros2 launch gazebo_ros gazebo.launch.py')
        print('Enter values then press Enter. Ctrl-C to quit.')
        while rclpy.ok():
            try:
                fx = float(input('Force X  (N): '))
            except ValueError:
                print('  not a number, try again')
                continue
            tz = float(input('Torque Z (Nm): '))
            msg = Wrench()
            msg.force.x = fx
            msg.torque.z = tz
            self.pub.publish(msg)
            print(f'  published force.x={fx:.1f}  torque.z={tz:.1f}\n')


def main():
    rclpy.init()
    node = ForceCmd()
    try:
        node.run()
    except KeyboardInterrupt:
        pass
    finally:
        node.destroy_node()
        rclpy.shutdown()


if __name__ == '__main__':
    main()
```

### `wheel_control/apply_wrench.py` — the subscriber

```python
import rclpy
from rclpy.node import Node
from builtin_interfaces.msg import Duration
from geometry_msgs.msg import Wrench
from gazebo_msgs.srv import ApplyLinkWrench, LinkRequest

PUSH_DURATION_SEC = 0.3  # how long each push lasts before we clear the wrench


class ApplyWrench(Node):
    """Subscribes to /wheel_cmd and pushes the wheel via /apply_link_wrench.

    NOTE: gazebo_ros has a known bug (#1540) where a wrench with a *positive*
    duration is never applied. The workaround is to apply it continuously
    (duration < 0) and then stop it with /clear_link_wrenches after a short time.
    """

    def __init__(self):
        super().__init__('apply_wrench')
        self.cli = self.create_client(ApplyLinkWrench, '/apply_link_wrench')
        self.clear_cli = self.create_client(LinkRequest, '/clear_link_wrenches')
        self.create_subscription(Wrench, '/wheel_cmd', self.cb, 10)
        self._clear_timer = None

    def wait_for_service(self, timeout_sec=30.0):
        self.get_logger().info('waiting for /apply_link_wrench service...')
        if self.cli.wait_for_service(timeout_sec=timeout_sec):
            self.get_logger().info('/apply_link_wrench service is ready')
            return True
        self.get_logger().warn(
            f'/apply_link_wrench service not found after {timeout_sec}s. '
            'Is Gazebo running? Try: ros2 launch gazebo_ros gazebo.launch.py')
        return False

    def clear_wrenches(self, *_):
        if self._clear_timer is not None:
            self._clear_timer.cancel()
            self._clear_timer = None
        if not self.clear_cli.service_is_ready():
            return
        req = LinkRequest.Request()
        req.link_name = 'unit_cylinder::link'
        self.clear_cli.call_async(req)
        self.get_logger().info('wrench cleared')

    def cb(self, msg):
        if not self.cli.service_is_ready():
            self.get_logger().info(
                'ignoring command: service not ready yet '
                '(command will work once Gazebo is up)')
            return

        # stop any previous push still in progress
        if self._clear_timer is not None:
            self._clear_timer.cancel()

        req = ApplyLinkWrench.Request()
        req.link_name = 'unit_cylinder::link'
        req.reference_frame = ''                 # empty = inertial frame
        req.wrench = msg                         # force + torque
        req.duration = Duration(sec=-1)          # < 0 => apply continuously (bug workaround)
        self.get_logger().info(
            f'applying force.x={msg.force.x}, torque.z={msg.torque.z} '
            f'for {PUSH_DURATION_SEC}s')
        self.cli.call_async(req)

        # clear the wrench after the push so the wheel coasts
        self._clear_timer = self.create_timer(PUSH_DURATION_SEC, self.clear_wrenches)


def main():
    rclpy.init()
    node = ApplyWrench()
    node.wait_for_service()
    node.clear_wrenches()          # best-effort: drop any stale wrench from before
    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        pass
    finally:
        node.destroy_node()
        rclpy.shutdown()


if __name__ == '__main__':
    main()
```

---

## 3. Build the package

```bash
source /opt/ros/humble/setup.bash
cd ~/Desktop/code/ros2_robotic/turtlebot3_ws
colcon build --packages-select wheel_control
source install/setup.bash
```

Check the executables exist:

```bash
ros2 pkg executables wheel_control
# wheel_control apply_wrench
# wheel_control force_cmd
```

---

## 4. Run the demo (4 terminals)

> **Only one gzserver at a time** — a second Gazebo launch will fail (port 11345 in
> use, exit 255). Close the old one before starting a new one.

**Terminal 1 — Gazebo:**
```bash
ros2 launch gazebo_ros gazebo.launch.py
```
Wait until `[gzserver-1]: process started` and no `process has died`.

**Terminal 2 — spawn the wheel:**
```bash
export GAZEBO_MODEL_PATH=$HOME/Desktop/code/ros2_robotic/models
ros2 run gazebo_ros spawn_entity.py -entity unit_cylinder \
  -file ~/Desktop/code/ros2_robotic/models/unit_cylinder/model.sdf -x 0 -y 0 -z 0
```
Expect: `Spawn status: SpawnEntity: Successfully spawned entity [unit_cylinder]`.

**Terminal 3 — subscriber:**
```bash
ros2 run wheel_control apply_wrench
```
Expect: `waiting for /apply_link_wrench service...` then `service is ready`.

**Terminal 4 — publisher (drive):**
```bash
ros2 run wheel_control force_cmd
```
Type a `Force X` then a `Torque Z`, press **Enter**. Repeat to push more. Ctrl-C to quit.

Start small — the wheel is 0.5 kg, so big numbers fling it far:
- Force X: `20`–`100` N (1500 N launched it ~2 km!)
- Torque Z: `5`–`50` Nm to spin it in place

---

## 5. Verify the pipeline

```bash
ros2 node list | grep -E "gazebo|apply_wrench|force_cmd"
ros2 service list | grep wrench          # /apply_link_wrench, /clear_link_wrenches
ros2 topic echo /wheel_cmd               # live view of what force_cmd publishes
```

Watch in the Gazebo GUI: one Enter should give one push, then the wheel coasts.

---

## 6. How it works (the full chain)

1. **`force_cmd`** (publisher) reads your numbers and publishes a
   `geometry_msgs/msg/Wrench` (`force` + `torque`) on topic **`/wheel_cmd`**.
2. **`apply_wrench`** (subscriber) receives it and calls the Gazebo ROS service
   **`/apply_link_wrench`** with `body = unit_cylinder::link`.
3. **Gazebo's `gazebo_ros_force_system` plugin** applies the wrench to the link,
   and the **ODE physics engine** moves the body.
4. Because of a known upstream bug (gazebo_ros_pkgs **#1540**: a wrench with a
   positive duration is never applied), we apply **continuously** (`duration = -1`)
   and **clear it after 0.3 s** via `/clear_link_wrenches` → a clean "push, then coast."

This is the same anatomy as the real TurtleBot3: `teleop → /cmd_vel → diff_drive
plugin → physics` — except here you replace the keyboard/plugin with your own
publisher and subscriber, and the "wheel" is a single free link instead of a robot.

---

## 7. Troubleshooting

| Symptom | Cause / Fix |
|---|---|
| `gzserver: process has died, exit code 255` | Another gzserver already owns port 11345. Close it first. |
| `service not found after 30s` | Gazebo isn't running, or wrong `ROS_DOMAIN_ID`. All terminals must share one domain (default 0). |
| Wheel spawns but floats/falls | Spawn with `-z 0` (model already places its link at z=0.15 internally). |
| Wheel won't roll | Collision must match visual (radius, `roll=1.5707`) and `kp` ≈ 1e5. See model.sdf above. |
| Huge force → wheel vanishes | The wheel is 0.5 kg. Use 20–100 N. |
