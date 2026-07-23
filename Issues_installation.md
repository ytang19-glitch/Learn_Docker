# ROS 2 Environment Leak from Host to Docker Container

## Problem

The Docker container was built for **ROS 2 Humble**, but ROS commands behaved as if **ROS 2 Jazzy** was being used.

Examples included:

```bash
echo $ROS_DISTRO
```

Unexpected output:

```text
jazzy
```

or

```bash
ros2 pkg list
```

showed packages from the host workspace instead of the container.

---

## Symptoms

Possible symptoms include:

- Wrong ROS distribution reported.
- Missing packages that should exist in Humble.
- `controller_manager` cannot be found.
- `franky_bridge` fails to launch.
- Package dependency conflicts.
- Unexpected library versions.
- Runtime errors caused by mixed environments.

---

## Root Cause

The host computer automatically sourced its ROS environment:

```bash
source /opt/ros/jazzy/setup.bash
```

and possibly:

```bash
source ~/franka_ros2_ws/install/setup.bash
```

When a shell inside the Docker container inherited these settings, the Humble environment became contaminated by the host Jazzy environment.

Conceptually:

```text
Host Ubuntu
ROS 2 Jazzy
        │
        ▼
Host .bashrc
(source /opt/ros/jazzy/setup.bash)
        │
        ▼
Docker Container (ROS 2 Humble)
        │
        ▼
Jazzy environment variables override
the Humble environment
        │
        ▼
ROS version conflict
```

---

## Why This Happens

ROS 2 relies on several environment variables, including:

```text
ROS_DISTRO
AMENT_PREFIX_PATH
COLCON_PREFIX_PATH
CMAKE_PREFIX_PATH
LD_LIBRARY_PATH
PYTHONPATH
PATH
```

If these variables reference the host workspace instead of the container, ROS resolves packages and libraries from the wrong installation.

---

## Solution

### Step 1

Check which ROS environments are sourced:

```bash
cat ~/.bashrc | grep ros
```

Remove or comment out:

```bash
source /opt/ros/jazzy/setup.bash
```

and

```bash
source ~/franka_ros2_ws/install/setup.bash
```

if they are inappropriate for the container.

---

### Step 2

Inside the Docker container, source only the correct ROS installation:

```bash
source /opt/ros/humble/setup.bash
```

---

### Step 3

If using a workspace:

```bash
cd ~/franka-docker/franka_ws

colcon build

source install/setup.bash
```

---

## Verification

Confirm the active ROS distribution:

```bash
echo $ROS_DISTRO
```

Expected output:

```text
humble
```

Verify the workspace:

```bash
ros2 pkg list | grep franky
```

The package should come from the container workspace rather than the host installation.

---

## Correct Workflow

```text
Docker Container
        │
        ▼
source /opt/ros/humble/setup.bash
        │
        ▼
Build workspace
        │
        ▼
source install/setup.bash
        │
        ▼
Launch ROS nodes
```

The host ROS environment should never override the container's environment.

---

## Lessons Learned

- Keep the Docker environment isolated from the host.
- Avoid automatically sourcing host ROS workspaces inside containers.
- Build the workspace before sourcing `install/setup.bash`.
- Always verify the active ROS distribution with `echo $ROS_DISTRO`.
