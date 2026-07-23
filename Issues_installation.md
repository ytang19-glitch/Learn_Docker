# Docker Startup, ROS 2 Workspace Sourcing, and Host Environment Leakage

## Overview

This document records three related issues encountered during the FR3 Docker environment setup:

1. Docker service failure caused by an inactive `docker.socket`.
2. Container startup failure caused by sourcing a ROS 2 workspace before it had been built.
3. ROS 2 Jazzy environment variables from the host leaking into a ROS 2 Humble container.

---

# Issue 1 — Docker Service Failed to Start

## Problem

Docker service failed to start with:

```text
failed to load listeners: no sockets found via socket activation:
make sure the service was started by systemd
```

Systemd logs showed:

```text
docker.service: Main process exited, code=exited, status=1/FAILURE
```

---

## Root Cause

Docker was configured to use systemd socket activation:

```text
-H fd://
```

This means:

- `dockerd` expects systemd to provide an already-open socket.
- The socket is managed by `docker.socket`.
- If `docker.socket` is missing, disabled, or not running, Docker cannot create its listener.
- The Docker daemon therefore exits immediately.

The failure was not caused by:

- Containers
- Images
- Storage corruption
- Networking
- Disk space

The specific cause was a broken or inactive `docker.socket`.

---

## How Docker Socket Activation Works

Systemd starts:

```text
docker.socket
```

This creates:

```text
/run/docker.sock
```

Systemd then launches:

```text
docker.service
```

and passes the socket file descriptor to `dockerd`.

If the socket is unavailable, Docker prints:

```text
failed to load listeners: no sockets found via socket activation
```

and exits with status code `1`.

---

## Fix

### Restart the socket and service

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker.socket
sudo systemctl restart docker.service
```

### Enable the socket permanently

```bash
sudo systemctl enable docker.socket
sudo systemctl start docker.socket
sudo systemctl start docker
```

---

## Verification

Check service status:

```bash
sudo systemctl status docker
sudo systemctl status docker.socket
```

Check Docker functionality:

```bash
docker ps
```

---

## Expected Working Configuration

The Docker service should contain:

```ini
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
```

The Docker socket should contain something similar to:

```ini
[Socket]
ListenStream=/run/docker.sock
SocketMode=0660
SocketUser=root
SocketGroup=docker
```

---

## Summary

Docker failed because `dockerd` was configured to receive its socket from systemd, but `docker.socket` was unavailable.

Restarting and enabling `docker.socket` restored the socket activation chain and allowed Docker to start normally.

---

# Issue 2 — ROS 2 Workspace Sourced Before Build

## Problem

The container reported:

```text
No such file or directory: install/setup.bash
```

The problem was caused by this Dockerfile instruction:

```dockerfile
RUN echo "source /home/user/franka-docker/franka_ws/install/setup.bash" >> ~/.bashrc
```

---

## Root Cause

This instruction forces every interactive shell to source the workspace setup file automatically.

The startup sequence becomes:

```text
Container starts
      │
      ▼
.bashrc is loaded
      │
      ▼
install/setup.bash is sourced
      │
      ▼
The workspace has not been built yet
      │
      ▼
install/setup.bash does not exist
      │
      ▼
Shell startup reports an error
```

The file may exist on the host, but that does not guarantee that it exists at the expected path inside the container.

A bind-mounted workspace may also hide files that were created in the image during the build.

---

## Correct Fix

### Recommended option: do not auto-source the workspace

Remove this line:

```bash
source /home/user/franka-docker/franka_ws/install/setup.bash
```

Build and source the workspace manually:

```bash
cd /home/user/franka-docker/franka_ws

colcon build

source install/setup.bash
```

This is the preferred approach because the container does not assume that the workspace has already been built.

---

### Optional safe auto-source configuration

If automatic sourcing is required, use a file-existence check:

```dockerfile
RUN echo 'if [ -f /home/user/franka-docker/franka_ws/install/setup.bash ]; then source /home/user/franka-docker/franka_ws/install/setup.bash; fi' >> ~/.bashrc
```

This prevents an error when the workspace has not yet been built.

---

## Correct ROS 2 and Docker Workflow

```text
1. Start the container
2. Source the base ROS 2 distribution
3. Enter the workspace
4. Build the workspace with colcon
5. Source install/setup.bash
6. Launch the ROS 2 nodes
```

Example:

```bash
source /opt/ros/humble/setup.bash

cd /home/user/franka-docker/franka_ws

rm -rf build install log

colcon build

source install/setup.bash
```

---

# Issue 3 — Host ROS 2 Jazzy Environment Leaks into Humble Container

## Problem

The host computer uses:

```text
Ubuntu 24.04
ROS 2 Jazzy
```

The Docker container uses:

```text
Ubuntu 22.04
ROS 2 Humble
```

Inside the container, ROS commands may unexpectedly use host paths, host workspace variables, or Jazzy-related settings.

Possible symptoms include:

- `echo $ROS_DISTRO` reports the wrong distribution.
- `AMENT_PREFIX_PATH` contains host Jazzy paths.
- `COLCON_PREFIX_PATH` contains host workspace paths.
- `PYTHONPATH` or `LD_LIBRARY_PATH` references host installations.
- Packages are resolved from the wrong workspace.
- `controller_manager`, `franky_bridge`, or other packages fail unexpectedly.
- Python imports load incompatible ROS 2 libraries.
- Humble binaries are mixed with Jazzy libraries.

---

## Root Cause

The host shell may automatically source:

```bash
source /opt/ros/jazzy/setup.bash
source ~/franka_ros2_ws/install/setup.bash
```

This populates environment variables such as:

```text
ROS_DISTRO
AMENT_PREFIX_PATH
COLCON_PREFIX_PATH
CMAKE_PREFIX_PATH
PYTHONPATH
LD_LIBRARY_PATH
PATH
```

Docker normally inherits environment variables passed by the launch command, Compose configuration, shell wrapper, or entrypoint.

A bind-mounted host home directory or copied host `.bashrc` can also cause the host ROS setup commands to run inside the container.

The resulting conflict looks like:

```text
Host PC: ROS 2 Jazzy
        │
        ▼
Host environment variables or .bashrc
        │
        ▼
Docker container: ROS 2 Humble
        │
        ▼
Jazzy and Humble paths become mixed
        │
        ▼
Package and library conflicts
```

---

## Important Correction

Because this container is designed for ROS 2 Humble, the expected value inside the container is:

```text
ROS_DISTRO=humble
```

It should not be:

```text
ROS_DISTRO=jazzy
```

The following command is correct only when `$ROS_DISTRO` is already set to `humble`:

```bash
source /opt/ros/$ROS_DISTRO/setup.bash
```

A safer explicit command is:

```bash
source /opt/ros/humble/setup.bash
```

---

## Diagnose the Environment

Inside the container, run:

```bash
echo "$ROS_DISTRO"
echo "$AMENT_PREFIX_PATH"
echo "$COLCON_PREFIX_PATH"
echo "$CMAKE_PREFIX_PATH"
echo "$PYTHONPATH"
echo "$LD_LIBRARY_PATH"
which ros2
```

Check for host-specific paths:

```bash
env | grep -E 'ROS|AMENT|COLCON|CMAKE_PREFIX|PYTHONPATH|LD_LIBRARY_PATH'
```

Check shell startup files:

```bash
grep -nE 'ros|colcon|setup.bash|franka_ros2_ws'     ~/.bashrc ~/.profile /etc/bash.bashrc 2>/dev/null
```

Expected ROS executable:

```text
/opt/ros/humble/bin/ros2
```

Expected distribution:

```text
humble
```

---

## Fix

### 1. Do not copy or mount the host `.bashrc`

Avoid mounting the host home directory over the container user's entire home directory.

Prefer mounting only the required workspace:

```bash
-v ~/franka-docker/franka_ws:/home/user/franka-docker/franka_ws
```

Do not use:

```bash
-v ~:/home/user
```

unless the implications are fully understood.

---

### 2. Do not pass host ROS variables into the container

Avoid Docker commands or Compose files that explicitly pass variables such as:

```text
ROS_DISTRO
AMENT_PREFIX_PATH
COLCON_PREFIX_PATH
CMAKE_PREFIX_PATH
PYTHONPATH
LD_LIBRARY_PATH
```

---

### 3. Start a clean shell for diagnosis

Use:

```bash
docker run -it --rm   --network host   --privileged   --entrypoint /bin/bash   fr3-docker   --noprofile --norc
```

Then source only Humble:

```bash
source /opt/ros/humble/setup.bash
```

---

### 4. Clear inherited ROS variables when necessary

Inside the container:

```bash
unset ROS_DISTRO
unset ROS_VERSION
unset ROS_PYTHON_VERSION
unset AMENT_PREFIX_PATH
unset COLCON_PREFIX_PATH
unset CMAKE_PREFIX_PATH
unset PYTHONPATH
```

Then source Humble:

```bash
source /opt/ros/humble/setup.bash
```

Do not blindly unset `PATH` or `LD_LIBRARY_PATH`; instead, start a clean shell or rebuild the container environment when these contain incompatible host paths.

---

### 5. Use a clean environment at container launch

A stricter diagnostic launch can use:

```bash
docker run -it --rm   --network host   --privileged   --env ROS_DISTRO=humble   --entrypoint /bin/bash   fr3-docker
```

For complete isolation, define only the required environment variables in the Dockerfile or entrypoint rather than forwarding the host environment.

---

### 6. Rebuild and source the container workspace

```bash
source /opt/ros/humble/setup.bash

cd /home/user/franka-docker/franka_ws

rm -rf build install log

colcon build

source install/setup.bash
```

---

## Verification

Run:

```bash
echo "$ROS_DISTRO"
which ros2
ros2 pkg prefix rclpy
ros2 pkg list | grep franky
```

Expected results:

```text
ROS_DISTRO: humble
ros2 executable: /opt/ros/humble/bin/ros2
```

The `franky` packages should resolve from the container workspace or the container installation, not from the host Jazzy workspace.

Check for remaining Jazzy paths:

```bash
env | grep -i jazzy
```

Expected result:

```text
No output
```

---

# Recommended Container Startup Flow

```text
Start isolated container
        │
        ▼
source /opt/ros/humble/setup.bash
        │
        ▼
Build container workspace
        │
        ▼
source install/setup.bash
        │
        ▼
Verify ROS_DISTRO and package paths
        │
        ▼
Launch franky_bridge
```

Example:

```bash
docker run -it --rm   --network host   --privileged   -v ~/franka-docker/franka_ws:/home/user/franka-docker/franka_ws   --entrypoint bash   fr3-docker
```

Inside the container:

```bash
source /opt/ros/humble/setup.bash

cd /home/user/franka-docker/franka_ws

colcon build

source install/setup.bash

echo "$ROS_DISTRO"

ros2 launch franky_ros franky_bringup.launch.py robot_ip:=172.16.0.2
```

---

# Final Verification Checklist

- [ ] `docker.socket` is active.
- [ ] `docker.service` is active.
- [ ] `docker ps` runs successfully.
- [ ] The container starts without workspace sourcing errors.
- [ ] The container reports `ROS_DISTRO=humble`.
- [ ] `which ros2` points to `/opt/ros/humble/bin/ros2`.
- [ ] No Jazzy paths appear in the container environment.
- [ ] The workspace builds successfully.
- [ ] `install/setup.bash` exists after the build.
- [ ] The workspace is sourced only after building.
- [ ] `franky_bridge` launches from the container environment.
