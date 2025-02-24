# ROS Docker Base Images

This repository provides lightweight and optimized Docker images for different ROS distributions:
- **ROS Noetic (Ubuntu 20.04)** → `achilleas2942/ros:noetic`
- **ROS Melodic (Ubuntu 18.04)** → `achilleas2942/ros:melodic`
- **ROS Humble (Ubuntu 22.04, ROS 2)** → `achilleas2942/ros:humble`

These images serve as **minimal base images** to create customized Docker containers for various robotic applications.

## 1. What's Included?
These images are built with **only essential dependencies**, making them lightweight and efficient:
✅ **Base ROS installation** (Core + essential communication libraries)  
✅ `build-essential` for compiling ROS packages  
✅ `python3-rosdep` for managing dependencies (Python 2 for Melodic)  
✅ `python3-pip` (or `python-pip` for Melodic) for Python package management  
✅ `curl`, `git`, `ca-certificates`, `gnupg`, `lsb-release` for secure package downloads  
✅ `udev` for USB device access (if needed)  
✅ **Catkin/Colcon workspace setup** (`catkin_ws` for ROS 1, `ros2_ws` for ROS 2)  

These images **do not include unnecessary tools** like Rviz, Gazebo, or large libraries unless explicitly required.

---

## 2. How to Use These Images as Base Images
To create a custom ROS-based image, use one of these as your base image in your `Dockerfile`:

```dockerfile
# Choose the appropriate base image
FROM achilleas2942/ros:noetic  # or :melodic, :humble

# Install additional dependencies if needed
RUN apt-get update && apt-get install --no-install-recommends -y \
    ros-noetic-navigation \
    && rm -rf /var/lib/apt/lists/*

# Copy entrypoint script
COPY entrypoint.sh /root/entrypoint.sh
RUN chmod +x /root/entrypoint.sh

# Set default command
CMD ["/root/entrypoint.sh"]
```

Then, build your custom image:
```bash
docker build -t my_ros_image .
```

Run the container:
```bash
docker run --rm -it my_ros_image
```

---

## 3. How to Add Only Necessary Dependencies
When extending the base image, you should **only install the specific dependencies required** by your ROS application. Instead of installing everything, follow these best practices:

1. **Use `--no-install-recommends`** to avoid unnecessary dependencies.
2. **Remove `apt` lists after installation** to reduce image size.
```dockerfile
RUN apt-get update && apt-get install --no-install-recommends -y \
    ros-noetic-tf2-ros ros-noetic-rviz \
    && rm -rf /var/lib/apt/lists/*
```
3. **Use `rosdep`** to install package dependencies automatically (recommended for large projects).
```dockerfile
COPY package.xml /root/catkin_ws/src/my_package/package.xml
RUN rosdep update && rosdep install --from-paths /root/catkin_ws/src --ignore-src -r -y
```

---

## 4. Using an Entrypoint Script for Dynamic Configuration
Instead of adding static commands in the `Dockerfile`, **use an `entrypoint.sh` script** to dynamically configure the container at runtime.

A typical **entrypoint script** can:
- ✅ Source the ROS environment
- ✅ Set environment variables like `ROS_MASTER_URI` and `ROS_IP`
- ✅ Clone or update repositories
- ✅ Build the workspace
- ✅ Run ROS launch files or other startup scripts

### Example `entrypoint.sh`
```bash
#!/bin/bash

# Source ROS setup
source /opt/ros/noetic/setup.bash

# Set ROS networking variables
ROS_MASTER_IP=${1:-$(hostname -I | awk '{print $1}')}
export ROS_MASTER_URI=http://$ROS_MASTER_IP:11311
export ROS_IP=$ROS_MASTER_IP

# Clone/update repositories
cd /root/catkin_ws/src
if [ ! -d "my_robot" ]; then
    git clone https://github.com/myorg/my_robot.git
else
    cd my_robot && git pull
fi

# Build the workspace
cd /root/catkin_ws && catkin_make
source /root/catkin_ws/devel/setup.bash

# Start ROS application
exec "$@"
```

### Using the Entrypoint Script
In your `Dockerfile`, copy the script and set it as the entrypoint:
```dockerfile
COPY entrypoint.sh /root/entrypoint.sh
RUN chmod +x /root/entrypoint.sh
ENTRYPOINT ["/root/entrypoint.sh"]
```

Now, when running the container, you can **override default behaviors**:
```bash
docker run --rm -it my_ros_image /bin/bash
```

Or run a ROS launch file dynamically:
```bash
docker run --rm -it my_ros_image roslaunch my_robot bringup.launch
```

---

## Summary
- ✅ **Use `FROM achilleas2942/ros:noetic` (or `melodic`, `humble`) as a base**  
- ✅ **Only install necessary dependencies to keep images lightweight**  
- ✅ **Use an entrypoint script to pull repositories, set up the workspace, and launch applications dynamically**  
