# AIC MuJoCo Scene

MuJoCo simulation environment for the AI for Industry Challenge. Contains the UR5e robot, Robotiq Hand-E gripper, task board with connectors, and SFP-SC cable — ready for policy development and training.

> **Note:** Evaluation runs in Gazebo. MuJoCo is for training and development only. Your final submission must work via standard ROS 2 topics.

## Getting Started

### 1. Clone and pull assets

```bash
git clone <repo-url>
cd fisher_boys
git lfs pull   # Downloads mesh/texture files from LFS
```

### 2. Source the AIC workspace

```bash
source ~/ws_aic/install/setup.bash
```

This sets `MUJOCO_PLUGIN_PATH` which is required for the cable elasticity plugin.

### 3. Launch with ROS 2 (Recommended)

```bash
# Terminal 1: Zenoh router
export RMW_IMPLEMENTATION=rmw_zenoh_cpp
export ZENOH_CONFIG_OVERRIDE='transport/shared_memory/enabled=true'
ros2 run rmw_zenoh_cpp rmw_zenohd

# Terminal 2: MuJoCo simulation
export RMW_IMPLEMENTATION=rmw_zenoh_cpp
export ZENOH_CONFIG_OVERRIDE='transport/shared_memory/enabled=true'
ros2 launch aic_mujoco aic_mujoco_bringup.launch.py

# Terminal 3: Teleop (optional)
export RMW_IMPLEMENTATION=rmw_zenoh_cpp
export ZENOH_CONFIG_OVERRIDE='transport/shared_memory/enabled=true'
ros2 run aic_teleoperation cartesian_keyboard_teleop
```

### Quick Python test

```python
import mujoco
import mujoco.viewer

model = mujoco.MjModel.from_xml_path('mujoco/models/scene.xml')
data = mujoco.MjData(model)

# Set home position (arm collapses without this)
home = {'shoulder_pan_joint': 0.0, 'shoulder_lift_joint': -1.57,
        'elbow_joint': 1.57, 'wrist_1_joint': -1.57,
        'wrist_2_joint': -1.57, 'wrist_3_joint': 0.0}

for name, val in home.items():
    jid = mujoco.mj_name2id(model, mujoco.mjtObj.mjOBJ_JOINT, name)
    data.qpos[model.jnt_qposadr[jid]] = val
for i, val in enumerate(home.values()):
    data.ctrl[i] = val

mujoco.mj_forward(model, data)
mujoco.viewer.launch(model, data)
```

## File Structure

```
mujoco/models/
├── scene.xml         # Load this — includes robot + world, sets integrator/timestep
├── aic_robot.xml     # UR5e + gripper with actuators, sensors, F/T sensor, cameras
├── aic_world.xml     # Enclosure, floor, task board, cable with elasticity plugin
└── assets/           # OBJ/STL meshes and PNG textures (tracked with git-lfs)
```

## Actuators

| Index | Name | Joint | Range |
|-------|------|-------|-------|
| 0 | shoulder_pan_joint_motor | shoulder_pan_joint | ±6.28 rad |
| 1 | shoulder_lift_joint_motor | shoulder_lift_joint | ±6.28 rad |
| 2 | elbow_joint_motor | elbow_joint | ±3.14 rad |
| 3 | wrist_1_joint_motor | wrist_1_joint | ±6.28 rad |
| 4 | wrist_2_joint_motor | wrist_2_joint | ±6.28 rad |
| 5 | wrist_3_joint_motor | wrist_3_joint | ±6.28 rad |
| 6 | gripper left_finger_motor | gripper left_finger_joint | 0–0.025 m |

Right gripper finger is coupled via mimic constraint — only control the left finger.

## Sensors

- **F/T:** `force_AtiForceTorqueSensor`, `torque_AtiForceTorqueSensor`
- **Joints:** `sensor_<joint>_pos` / `sensor_<joint>_vel` for all 6 UR5e joints

## Scene Components

- **UR5e** 6-DOF arm with Robotiq Hand-E gripper
- **Axia80 F/T sensor** on wrist
- **3 Basler cameras** (center, left, right)
- **Task board** with SC mount, SFP mount, NIC card mount, SC port
- **SFP-SC cable** with `mujoco.elasticity.cable` plugin
- **LC plug** welded to gripper, **SC plug** on cable end

## Cable Note

The cable requires the MuJoCo elasticity plugin (`libelasticity.so`). If the cable explodes, make sure you've sourced the workspace (`source ~/ws_aic/install/setup.bash`) so that `$MUJOCO_PLUGIN_PATH` is set correctly.