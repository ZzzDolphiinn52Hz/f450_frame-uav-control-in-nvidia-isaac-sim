# F450 Isaac Sim

File nay gom cac lenh can chay theo dung thu tu. Ban chi can copy tung block vao Isaac Sim Script Editor.

## 1. Mo Isaac Sim

Chay trong terminal:

```bash
conda init
conda deactivate
conda activate env_isaaclab
issacsim
```

Sau khi Isaac Sim mo len, load scene co drone `/f450_simple/base_link`, sau do mo `Window > Script Editor`.

## 2. Chay Controller Chinh

Copy toan bo block nay vao Isaac Sim Script Editor va nhan Run.

```python
import sys
import importlib
import os

# Neu may ban dung duong dan khac, chi can sua dong nay.
CONTROLLER_PATH = "/config/Desktop/IssacSim_TA/f450/ros2_ws/src/f450_description/src"
SCRIPT_PATH = CONTROLLER_PATH + "/scripr_editor"
DATA_DIR = CONTROLLER_PATH + "/data"
TRACKING_CSV = DATA_DIR + "/f450_tracking.csv"

for path in (CONTROLLER_PATH, SCRIPT_PATH):
    if path not in sys.path:
        sys.path.insert(0, path)

os.makedirs(DATA_DIR, exist_ok=True)
importlib.invalidate_caches()

# Dung controller cu neu da Run truoc do.
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

    # Neu khong khai bao x_target/y_target, drone se giu vi tri XY hien tai.
    # x_target=0.0,
    # y_target=0.0,
    z_target=1.0,

    # Gia tri da tune tu file tracking gan day.
    pwm_hover=1650.0,
)

# Gioi han goc nghieng do bo dieu khien XY tao ra.
f450_app.position_angle_limit_deg = 6.5
f450_app.position_accel_limit = 1.2

# Dat yaw mong muon, don vi radian. 0.0 la giu huong ban dau theo truc world.
f450_app.set_yaw_target(0.0)

# Bat log de ve do thi tracking.
f450_app.start_tracking_log(TRACKING_CSV, sample_period=0.02)

# Bat dau dieu khien.
f450_app.start()

print("F450 controller STARTED")
print("Tracking CSV:", TRACKING_CSV)
```

## 3. Doi Vi Tri Dat Khi Dang Chay

Sau khi block 2 da chay xong, neu muon doi vi tri dat thi copy block nay vao Script Editor va Run.

```python
# Dat vi tri moi: x, y, z theo met.
f450_app.set_xyz_target(
    x_target=1.0,
    y_target=0.0,
    z_target=1.5,
)

# Dat yaw moi, don vi radian.
f450_app.set_yaw_target(0.0)

print("New target: x=1.0, y=0.0, z=1.5, yaw=0.0 rad")
```

Vi du khac:

```python
f450_app.set_xyz_target(0.0, 0.0, 1.0)
f450_app.set_yaw_target(1.57)
```

## 4. Chay Quy Dao Hinh Tron

Neu chi muon giu vi tri, bo qua muc nay.

Sau khi block 2 da chay xong, copy block nay vao Script Editor va Run.

```python
import circle_trajectory as ct
importlib.reload(ct)

# Sua thong so quy dao o day neu muon.
ct.CIRCLE_RADIUS = 2.0
ct.CIRCLE_Z = 1.5
ct.CIRCLE_PERIOD = 20.0
ct.CIRCLE_CENTER_X = 0.0
ct.CIRCLE_CENTER_Y = 0.0
ct.TAKEOFF_TIME = 5.0
ct.LOOKAHEAD_DT = 0.18

# Dung chung file log voi controller chinh.
ct.TRACKING_CSV = TRACKING_CSV

ct.start(f450_app)
```

Co the Run lai block tren neu muon doi thong so circle. Script se tu go callback cu de tranh loi `RecursionError`.

De dung quy dao hinh tron va luu log:

```python
import circle_trajectory as ct
ct.stop(f450_app)
```

## 5. Dung Controller Va Dong File Log

Khi khong chay nua, copy block nay vao Script Editor va Run.

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

## 6. Ve Do Thi Sau Khi Chay

Chay cac lenh nay trong terminal, tai thu muc project `issacsim-practice-byme`.

Ve tracking 6 thanh phan `x, y, z, roll, pitch, yaw`:

```bash
python3 f450/ros2_ws/src/f450_description/src/data/plot_tracking.py f450/ros2_ws/src/f450_description/src/data/f450_tracking.csv --show
```

Ve live realtime khi Isaac Sim dang ghi CSV:

```bash
python3 f450/ros2_ws/src/f450_description/src/data/live_plot_tracking.py f450/ros2_ws/src/f450_description/src/data/f450_tracking.csv --wait --window 20
```

Ve quy dao 3D:

```bash
python3 f450/ros2_ws/src/f450_description/src/data/plot_trajectory_3d.py f450/ros2_ws/src/f450_description/src/data/f450_tracking.csv --show
```

## Ghi Chu Nhanh

- Block 2 la block quan trong nhat, dung de khoi dong controller.
- Block 3 dung de doi vi tri dat thu cong.
- Block 4 dung de bay theo hinh tron.
- Block 5 nen chay truoc khi tat Isaac Sim de dong file CSV.
- Neu Run lai block 2, controller cu se duoc stop truoc roi moi tao controller moi.
- Neu Isaac Sim da bao `RecursionError` o `circle_trajectory.py`, chay lai block 2 mot lan de tao controller sach, sau do moi chay lai block 4.
