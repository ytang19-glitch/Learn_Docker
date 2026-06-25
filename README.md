# Learn_Docker


https://github.com/ualberta-robotics/franka-docker/tree/main

Save the current user id into a file:

echo -e "USER_UID=$(id -u $USER)\nUSER_GID=$(id -g $USER)" > .env
It is needed to mount the folder from inside the Docker container.

Build the container:

docker compose build
Run the container:

docker compose up -d
Open a shell inside the container:

docker exec -it franka_ros2 /bin/bash
Clone the latests dependencies:

vcs import src < src/franka.repos --recursive --skip-existing
Build the workspace:

colcon build --symlink-install --cmake-args -DCMAKE_BUILD_TYPE=Release
Source the built workspace:

source install/setup.bash
When you are done, you can exit the shell and delete the container:

docker compose down -t 0
