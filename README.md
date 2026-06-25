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

# Build the FR3 Docker Image

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

---

# Run the Docker Container

Launch the container:

```bash
docker run -it --rm \
    --network host \
    --privileged \
    -v ~/franka_ros2_ws:/home/user/franka_ros2_ws \
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

