# Learn_Docker

```text
https://github.com/ualberta-robotics/franka-docker/tree/main
```
Save the current user id into a file:

echo -e "USER_UID=$(id -u $USER)\nUSER_GID=$(id -g $USER)" > .env
It is needed to mount the folder from inside the Docker container.
# Docker Installation and FR3 Verification Environment

This section documents the installation of Docker and the setup of the FR3 Docker environment used to independently verify the ROS 2 communication pipeline.

---

# Objective

The goal is to use the provided **ROS 2 Humble Docker container** as an isolated verification environment while keeping the primary development environment on **Ubuntu 24.04 + ROS 2 Jazzy**.

Docker provides an independent software stack, allowing comparison between two environments:

```text
Host PC
├── Ubuntu 24.04
├── ROS 2 Jazzy (Main Development)
└── Docker
      └── ROS 2 Humble (Verification)
```

If the Docker environment works while the native Jazzy environment does not, the issue is likely related to software compatibility rather than hardware or networking.

---

# Step 1. Update the System

```bash
sudo apt update
sudo apt upgrade -y
```

---

# Step 2. Install Required Packages

```bash
sudo apt install -y \
    ca-certificates \
    curl \
    gnupg
```

These packages are required to securely download and verify Docker packages.

---

# Step 3. Add Docker's Official GPG Key

Create the keyring directory:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
```

Download Docker's signing key:

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

Set the correct permissions:

```bash
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

---

# Step 4. Add the Docker Repository

```bash
echo \
"deb [arch=$(dpkg --print-architecture) \
signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

This adds Docker's official package repository to Ubuntu.

---

# Step 5. Install Docker

Update the package index:

```bash
sudo apt update
```

Install Docker and related tools:

```bash
sudo apt install -y \
    docker-ce \
    docker-ce-cli \
    containerd.io \
    docker-buildx-plugin \
    docker-compose-plugin
```

---

# Step 6. Verify the Installation

Check the Docker version:

```bash
docker --version
```

Example output:

```text
Docker version 28.x.x
```

Check that the Docker service is running:

```bash
sudo systemctl status docker
```

Expected output:

```text
Active: active (running)
```
if not:

```text
sudo systemctl daemon-reload
sudo systemctl restart docker.socket
sudo systemctl restart docker.service
```


---

# Step 7. Enable Docker Without sudo

Add the current user to the Docker group:

```bash
sudo usermod -aG docker $USER
```

Reload the group permissions:

```bash
newgrp docker
```

Verify:

```bash
docker run hello-world
```

Expected output:

```text
Hello from Docker!
```

This confirms that Docker is installed and functioning correctly.

---

# Step 8. Verify Docker Buildx

```bash
docker buildx version
```

Buildx provides advanced image-building functionality and is recommended for modern Docker workflows.

---

# Step 9. Build the FR3 Docker Image

Clone the repository:

```bash
git clone https://github.com/ualberta-robotics/franka-docker.git
```

Enter the repository:

```bash
cd franka-docker
```

Build the Docker image:

```bash
docker build -t fr3-docker .
```

# Step 10. Vertify the successful installation of docker image

```bash
sudo docker images
```

# Mscs: Docker Build Troubleshooting

If fail to build doker image:

## Build the Docker Image

Run the following command in the directory containing the `Dockerfile`:

```bash
docker build -t fr3-docker .
```

* `-t fr3-docker` assigns the image name.
* `.` uses the current directory as the build context.

---

## If the Build Fails

Rebuild with detailed logs:

```bash
sudo docker build -t fr3-docker -f Dockerfile . --progress=plain
```

This displays the full build output, making it easier to identify the step that failed.

---

## Common Issue: Intel RealSense
### Delete useless code 
If the build fails while accessing `librealsense.intel.com`, the RealSense repository may not support the Ubuntu version used in the Docker image.

### Injection from shell environment into docker
ROS2 startup script problem caused by the shell environment being injected into Docker

* Wrong configuration
```bash
Host (ROS2 Jazzy)
        │
        ▼
Host .bashrc
(source /opt/ros/jazzy/setup.bash)
        │
        ▼
Docker Container (ROS2 Humble)
        │
        ▼
Jazzy environment is injected
into the Humble container
        │
        ▼
ROS2 Version Conflicthost environment
```
* Correct configuration
```bash
  Docker Container (ROS2 Humble)
        │
        ▼
source /opt/ros/humble/setup.bash
        │
        ▼
Use Humble environment only
        │
        ▼
Container remains isolated
        │
        ▼
```
ROS2 Works Correctly
### Solution

1. Check the Ubuntu version in the `Dockerfile`:

```dockerfile
FROM ubuntu:22.04
```

2. If your project does **not** use an Intel RealSense  camera, remove or comment out the RealSense installation and repository setup in the `Dockerfile`.

---

3 Remove ROS 2 Workspace from `.bashrc`
source ~/franka_ros2_ws/install/setup.bash

Comment out or remove the following line from your host's `.bashrc`:

```bash
source ~/franka_ros2_ws/install/setup.bash
```

## Troubleshooting Checklist

* Build with `--progress=plain`.
* Identify the first failing command.
* Verify the Ubuntu version in the `Dockerfile`.
* Remove unnecessary dependencies (e.g., RealSense) if they are not required.
* Rebuild the Docker image.


```bash
https://github.com/ualberta-robotics/franka-docker/blob/main/Dockerfile
```
find:
```bash
#mkdir -p /etc/apt/keyrings && \
     curl -sSf https://librealsense.intel.com/Debian/librealsense.pgp | \
         tee /etc/apt/keyrings/librealsense.pgp > /dev/null && \
```
* debug source setup.bash
  observe controller_manager
  check environment variable
  
  open terminal inside container

 ```bash 
  sudo docker run -it --rm \
  --network host \
  --privileged \
  --entrypoint bash \
  fr3-docker
```
* Rebuild it after any change on dockerfile
  
As we run the container, Docker is still using:the OLD image
(which still contains source setup.bash and the old configuration)

Note:
 Dockerfile → docker build
 bashrc → source ~/.bashrc
  
```bash
cd ~/franka-docker
```
```bash
sudo docker build --no-cache -t fr3-docker .
```

# Steps 11 Run the Docker Container

Launch the container:

Container starts robot automatically 
```bash
sudo docker run -it --rm \
     --network host \
     --privileged \
     -v ~/franka-docker/franka_ws:/home/user/franka-docker/franka_ws \
     fr3-docker
```
or safe way
Container starts terminal instead (--entrypoint bash )
```bash
sudo docker run -it --rm \
  --network host \
  --privileged \
  -v ~/franka-docker/franka_ws:/home/user/franka-docker/franka_ws \
  --entrypoint bash \
  fr3-docker
```

Parameter explanation:

| Option           | Description                                                        |
| ---------------- | ------------------------------------------------------------------ |
| `-it`            | Interactive terminal                                               |
| `--rm`           | Remove container after exit                                        |
| `--network host` | Share the host network, allowing direct communication with the FR3 |
| `--privileged`   | Grant access to hardware interfaces                                |
| `-v`             | Mount the local ROS 2 workspace inside the container               |

---

# Step 12. start robot driver

Check if the docker(humble) is injetced by the host(jazzy)
```bash
cat ~/.bashrc | grep ros
```
rebuild workplace of docker:
```bash
cd /home/user/franka-docker/franka_ws
rm -rf build install log
colcon build
```
make sure franky is available
```bash
ros2 pkg list | grep franky
```
if certificate of ssl is expired:
```bash
cd /usr/local/lib/python3.10/dis-packages?panda_py
nano __init__/py
```

"ssl_context" like a security rule card we give to the websocket connection.

```bash
import ssl
ssl_context = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)

# Disable certificate verification
_ssl_context.check_hostname = False
_ssl_context.verify_mode = ssl.CERT_NONE
ssl=ssl_context,
```

franka_bridge is a communication layer between ROS 2 and the real Franka robot.
```bash
ros2 launch franky_ros franky_bringup.launch.py
```
Note:
the wire network should align with subnet of configuration
https://franka.de/hubfs/Hardware%20Manual%20Franka%20Research%203_Arm%20v2.1_R02210_1.3_EN.pdf?hsLang=en (p33)

```bash
changeing the wire changes the phsical connection, changing ip configuration changes
the logical network
For robot , both the phsical links and ip subnet must match before communication works 
```

# Verification Purpose

The Docker environment is used solely to verify the complete FR3 communication pipeline:

```text
PC Network
      │
      ▼
FCI Connection
      │
      ▼
libfranka
      │
      ▼
franka_hardware
      │
      ▼
controller_manager
      │
      ▼
ROS 2 Controllers
      │
      ▼
FR3 Robot
```

This helps isolate whether the issue originates from:

* ROS 2 Jazzy
* ros2_control
* franka_ros2
* libfranka
* Hardware communication

rather than the physical robot itself.

---

# Expected Outcome

If the Docker (ROS 2 Humble) environment successfully launches the controller while the native Ubuntu 24.04 + ROS 2 Jazzy environment does not, the problem is likely related to:

* ROS 2 Jazzy compatibility
* ros2_control version mismatch
* libfranka version mismatch
* franka_ros2 branch incompatibility

If both environments exhibit the same issue, the focus should shift to:

* FCI communication
* Network configuration
* System Image compatibility
* Robot hardware initialization

Using Docker as an independent verification environment provides a reliable method to distinguish software compatibility issues from hardware or network-related problems.

