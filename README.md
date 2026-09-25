# F450 Quadrotor Control in NVIDIA Isaac Sim

Hands-on notes for running a cascaded PID controller on an F450 quadrotor inside Isaac Sim.

Copy each Python block into **Window → Script Editor** and click **Run**. Run the sections in order.

| Step | What it does |
| --- | --- |
| 1 | Launch Isaac Sim and load the F450 scene |
| 2 | Start the main controller and tracking log |
| 3 | Change the XYZ / yaw setpoint while flying |
| 4 | Optional circular trajectory |
| 5 | Stop the controller and close the CSV log |
| 6 | Plot tracking from a terminal |

---

## Requirements

- NVIDIA Isaac Sim (launched from the `env_isaaclab` conda environment)
- Scene prim: `/f450_simple/base_link`
- Local checkout of this repository

Edit `CONTROLLER_PATH` in Section 2 if your machine uses a different folder.

> The module names `issac_attitude_hold` and the folder `scripr_editor` match the current tree. Do not rename them in the Script Editor blocks unless you also rename the files.

---

## 1. Launch Isaac Sim

```bash
conda init
conda deactivate
conda activate env_isaaclab
issacsim
```

When the app opens:

1. Load the scene that contains `/f450_simple/base_link`.
2. Open **Window → Script Editor**.

---

## 2. Start the main controller

Paste the entire block into the Script Editor and **Run**.

```python
import sys
import importlib
import os

# Change this if the repo lives somewhere else on your machine.
CONTROLLER_PATH = "/config/Desktop/IssacSim_TA/f450/ros2_ws/src/f450_description/src"
SCRIPT_PATH = CONTROLLER_PATH + "/scripr_editor"
DATA_DIR = CONTROLLER_PATH + "/data"
TRACKING_CSV = DATA_DIR + "/f450_tracking.csv"

for path in (CONTROLLER_PATH, SCRIPT_PATH):
    if path not in sys.path:
        sys.path.insert(0, path)

os.makedirs(DATA_DIR, exist_ok=True)
importlib.invalidate_caches()

# Stop a previous instance if this block was already run.
try:
    f450_app.stop()
except Exception:
    pass

MODULE_NAMES = [
    "f450_controller.control_utils",
    "f450_controller.motor_model",
    "f450_controller.altitude_hold",
    "f450_controller.attitude_pid",
    "f450_controller.position_hold",
    "f450_controller.yaw_hold",
    "f450_controller.motor_mixer",
    "f450_controller.propeller_spinner",
    "f450_controller.tracking_logger",
    "f450_controller.attitude_hold_compat",
    "f450_controller.issac_attitude_hold",
]

modules = {}
for module_name in MODULE_NAMES:
    modules[module_name] = importlib.import_module(module_name)

for module_name in MODULE_NAMES:
    modules[module_name] = importlib.reload(modules[module_name])

issac_attitude_hold = modules["f450_controller.issac_attitude_hold"]

f450_app = issac_attitude_hold.F450AttitudeHold(
    base_link_path="/f450_simple/base_link",
    # Leave x_target / y_target unset to hold the current XY position.
    # x_target=0.0,
    # y_target=0.0,
    z_target=1.0,
    # Hover PWM tuned from recent tracking logs.
    pwm_hover=1650.0,
)

# Limit the tilt commanded by the XY position loop.
f450_app.position_angle_limit_deg = 6.5
f450_app.position_accel_limit = 1.2

# Yaw target in radians. 0.0 keeps the initial world-frame heading.
f450_app.set_yaw_target(0.0)

# Enable CSV logging for later plots.
f450_app.start_tracking_log(TRACKING_CSV, sample_period=0.02)

f450_app.start()

print("F450 controller STARTED")
print("Tracking CSV:", TRACKING_CSV)
```

This block is required. Everything after it assumes `f450_app` is already running.

---

## 3. Change the setpoint at runtime

After Section 2 has started, paste and **Run**:

```python
# Position targets in metres.
f450_app.set_xyz_target(
    x_target=1.0,
    y_target=0.0,
    z_target=1.5,
)

# Yaw target in radians.
f450_app.set_yaw_target(0.0)

print("New target: x=1.0, y=0.0, z=1.5, yaw=0.0 rad")
```

Other examples:

```python
f450_app.set_xyz_target(0.0, 0.0, 1.0)
f450_app.set_yaw_target(1.57)
```

---

## 4. Circular trajectory (optional)

Skip this section if you only want position hold.

After Section 2 has started, paste and **Run**:

```python
import circle_trajectory as ct
importlib.reload(ct)

# Trajectory parameters (edit as needed).
ct.CIRCLE_RADIUS = 2.0
ct.CIRCLE_Z = 1.5
ct.CIRCLE_PERIOD = 20.0
ct.CIRCLE_CENTER_X = 0.0
ct.CIRCLE_CENTER_Y = 0.0
ct.TAKEOFF_TIME = 5.0
ct.LOOKAHEAD_DT = 0.18

# Reuse the same log file as the main controller.
ct.TRACKING_CSV = TRACKING_CSV

ct.start(f450_app)
```

Re-run the same block to change circle parameters. The script removes the previous callback first so you do not hit `RecursionError`.

Stop the circle and flush the log:

```python
import circle_trajectory as ct
ct.stop(f450_app)
```

If Isaac Sim already raised `RecursionError` inside `circle_trajectory.py`, run **Section 2** again to create a clean controller, then run Section 4.

---

## 5. Stop the controller

Run this before closing Isaac Sim so the CSV is closed cleanly.

```python
try:
    f450_app.stop_tracking_log()
except Exception:
    pass

try:
    f450_app.stop()
except Exception:
    pass

print("F450 controller STOPPED")
```

Re-running Section 2 also stops the previous instance before creating a new one.

---

## 6. Plot results

From the repository root (`issacsim-practice-byme`):

**6-DOF tracking** (`x, y, z, roll, pitch, yaw`):

```bash
python3 f450/ros2_ws/src/f450_description/src/data/plot_tracking.py \
  f450/ros2_ws/src/f450_description/src/data/f450_tracking.csv --show
```

**Live plot** while Isaac Sim is writing the CSV:

```bash
python3 f450/ros2_ws/src/f450_description/src/data/live_plot_tracking.py \
  f450/ros2_ws/src/f450_description/src/data/f450_tracking.csv --wait --window 20
```

**3D trajectory**:

```bash
python3 f450/ros2_ws/src/f450_description/src/data/plot_trajectory_3d.py \
  f450/ros2_ws/src/f450_description/src/data/f450_tracking.csv --show
```

---

## Notes

- Section 2 starts the controller. Do not skip it.
- Section 3 retargets XYZ / yaw by hand.
- Section 4 overlays a circular reference; Section 5 shuts everything down.
- Always run Section 5 before quitting Isaac Sim, or the tracking CSV may be incomplete.
- Controller loops: altitude, attitude, horizontal position, and yaw, with a custom motor mixer and propeller spin.
- Hover PWM (`pwm_hover=1650`) and tilt limits (`6.5 deg`, `1.2 m/s²`) were tuned from recent logs; change them if the vehicle or mass is different.
