# Python Environment Setup

## Install pip

```bash
sudo apt install python3-pip

# Verify installation
python3 -m pip --version
```

---

## Install Virtual Environment Support

```bash
sudo apt install python3-venv
```

---

## Create a Virtual Environment

```bash
python3 -m venv myenv
```

Activate the environment:

```bash
source myenv/bin/activate
```

Install Python packages:

```bash
pip install requests
```

Exit the virtual environment:

```bash
deactivate
```

---

# Verify Python Package Location

To check where a Python package is installed:

```bash
python3 -c "import panda_py; print(panda_py.__file__)"
```

Example output:

```text
/usr/local/lib/python3.10/dist-packages/panda_py/__init__.py
```

Open the file if modifications are required:

```bash
nano /usr/local/lib/python3.10/dist-packages/panda_py/__init__.py
```

---

# Franky ROS Bridge

The bridge connects the Franka FR3 hardware to ROS 2.

## Initialization Parameters

```python
class FrankyRosBridge(Node):
    def __init__(self):
        super().__init__("franky_ros_bridge")

        self.declare_parameter("robot_ip", "192.168.0.1")
        self.declare_parameter("realtime", False)
        self.declare_parameter("max_velocity", 0.05)
        self.declare_parameter("max_acceleration", 0.1)
        self.declare_parameter("max_jerk", 0.15)

        robot_ip = self.get_parameter(
            "robot_ip"
        ).get_parameter_value().string_value

        realtime = self.get_parameter(
            "realtime"
        ).get_parameter_value().bool_value

        max_vel = self.get_parameter(
            "max_velocity"
        ).get_parameter_value().double_value

        max_acc = self.get_parameter(
            "max_acceleration"
        ).get_parameter_value().double_value

        max_jerk = self.get_parameter(
            "max_jerk"
        ).get_parameter_value().double_value
```

---

## Desk Connection

```python
self.get_logger().info(
    f"Connecting to FR3 and Gripper at {robot_ip}..."
)

self.desk = panda_py.Desk(
    robot_ip,
    "Tangyujie",
    "tangyujie2003",
    platform="fr3",
)

self.desk.take_control(force=True)

self.get_logger().info(
    str(self.desk.has_control())
)

self.desk.unlock()
self.desk.activate_fci()
```

---

## Hardware Connection

```python
self.get_logger().info(
    "Establishing hardware connection..."
)

try:

    rt_conf = (
        franky.RealtimeConfig.Enforce
        if realtime
        else franky.RealtimeConfig.Ignore
    )

    self.robot = franky.Robot(
        robot_ip,
        realtime_config=rt_conf,
    )

    # Currently fixed.
    # RelativeDynamicsFactor API is not behaving as expected.
    self.robot.relative_dynamics_factor = 0.05

    self.gripper = franky.Gripper(robot_ip)

    self.get_logger().info(
        "Hardware connection established."
    )

    self.get_logger().info(
        f"Gripper max width: {self.gripper.max_width}"
    )

except Exception as e:

    self.get_logger().error(
        f"Failed to connect to robot: {e}"
    )

    traceback.print_exc()
    raise
```

---

# Common Connection Error

## Error Message

```text
[INFO] Establishing hardware connection...

[ERROR] Failed to connect to robot:

libfranka: Connection timeout.
Please check your network connection or settings.
```

Full traceback:

```text
franky._franky.NetworkException:
libfranka: Connection timeout.
Please check your network connection or settings.
```

---

# Cause

The robot cannot establish a **libfranka** connection.

Typical causes include:

- Incorrect robot IP address
- Ethernet cable connected to the wrong port
- FCI not activated
- PC network adapter not configured correctly
- Robot and workstation are not on the same subnet
- Firewall blocking communication
- Robot is not in the correct operating state

---

# Example Configuration

```text
Robot IP
192.168.0.1
```

---

# Debugging Checklist

Before running the Franky ROS Bridge, verify:

- [ ] Ethernet cable is connected to the FR3 Control interface.
- [ ] Workstation IP is configured correctly.
- [ ] Robot IP matches the bridge parameter.
- [ ] FCI has been activated from Franka Desk.
- [ ] The robot is unlocked.
- [ ] The workstation can ping the robot.
- [ ] No other application is controlling the robot.
- [ ] libfranka version is compatible with the installed Franky package.

---

# Expected Successful Output

```text
Connecting to FR3 and Gripper...

Taking control...

Unlocking robot...

Activating FCI...

Establishing hardware connection...

Hardware connection established.

Gripper max width: ...
```
