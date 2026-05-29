# Abaja 2026 Endurance System

This repository contains the integrated Autonomous Driving stack for the Abaja 2026 **Endurance** capstone event in IPG CarMaker.

The Endurance event requires full autonomous control of both lateral and longitudinal motion under a sequential, multi-phase driving challenge. This codebase merges the **Lane Keeping Assist (LKA)** and **Autonomous Emergency Braking (AEB)** systems into a single, unified ROS2 Humble workspace.

## System Architecture

The core of the system is the **Unified CarMaker Bridge**, which communicates with CarMaker over a TCP socket.

1.  **Lateral Control (LKA):**
    *   **Sensors:** RSDS Camera stream (`rsds_camera_publisher`).
    *   **Perception:** Ultra-Fast-Lane-Detection-v2 (UFLDv2) (`lane_detection_node`).
    *   **Planning/Control:** Stanley Controller (`stanley_controller_node`) computes the required steering angle.
    *   **Actuation:** The unified bridge writes the steering angle to `VC.Steer.Ang` via Direct Variable Access (DVA).

2.  **Longitudinal Control (AEB & Cruise):**
    *   **Sensors:** The unified bridge directly reads `Sensor.Radar.Vhcl.RAD00` object distances/velocities and `Vhcl.v` ego speed.
    *   **Perception & Control:** A Fuzzy Logic controller (`aeb_fuzzy_node`) calculates required braking force.
    *   **Actuation:**
        *   If the AEB node signals an emergency, the bridge takes full longitudinal control, forces throttle to 0, and applies the brake.
        *   If no emergency is present, the bridge uses a basic cruise controller to maintain a 10 km/h speed limit.

## Prerequisites

*   **OS:** Ubuntu 22.04
*   **ROS2:** Humble Hawksbill
*   **Simulator:** IPG CarMaker 11+
*   **Python:** 3.10+

### Dependencies

Ensure you have the required Python packages for deep learning and image processing:

```bash
pip install torch torchvision opencv-python numpy scipy gdown
```

## Setup & Installation

1.  **Clone the workspace:**
    Ensure this directory structure exists at your desired location (e.g., `~/endurance_ws`).

2.  **Download UFLDv2 Weights:**
    The lane detection node requires the pre-trained ResNet34 weights.
    ```bash
    cd ~/endurance_ws
    mkdir -p Ultra-Fast-Lane-Detection-v2/weights
    gdown "1AjnvAD3qmqt_dGPveZJsLZ1bOyWv62Yj" -O Ultra-Fast-Lane-Detection-v2/weights/culane_res34.pth
    ```

3.  **Build the Workspace:**
    ```bash
    cd ~/endurance_ws
    source /opt/ros/humble/setup.bash
    colcon build --symlink-install
    ```

## Running the Simulation

1.  **Start CarMaker:**
    *   Launch IPG CarMaker.
    *   Load your target Endurance TestRun.
    *   **Important:** Start the simulation *before* launching the ROS2 nodes to ensure the TCP sockets are open and the Data Dictionary (Quantities) are populated.

2.  **Launch the Endurance Stack:**
    Open a terminal and run the unified launch file:

    ```bash
    cd ~/endurance_ws
    source /opt/ros/humble/setup.bash
    
    # Required for the lane detection node to find the model scripts
    export UFLDV2_DIR=$(pwd)/Ultra-Fast-Lane-Detection-v2
    
    ros2 launch endurance endurance_system.launch.py
    ```

## Launch File Components (`endurance_system.launch.py`)

The launch script orchestrates the following nodes:

*   **`carmaker_complete_interface`**: Connects to `172.23.128.1:16660`. Reads radar/speed, writes steering/gas/brake.
*   **`aeb_fuzzy_node`**: Subscribes to radar topics, publishes `/aeb/brake_cmd`.
*   **`rsds_camera_publisher`**: Connects to CarMaker's RSDS server (port 2210), publishes ROS2 `Image` messages.
*   **`lane_detection_node`**: Loads UFLDv2, consumes images, publishes lane centers.
*   **`stanley_controller_node`**: Subscribes to lane centers, publishes steering commands to `/vehicle_control`.
*   **`vehicle_control_node`**: Relays Stanley commands to the interface.
*   **`performance_logger_node`**: Logs RMSE and frame rates to `lka_logs/`.
*   **`showimage`**: Opens a live visualizer for the UFLDv2 detection.

## Troubleshooting

*   **"CarMaker connection failed / Connection refused"**
    *   Ensure CarMaker is running *and* the simulation is actively playing before starting the ROS2 launch file.
*   **"executable 'rsds_camera_publisher' not found"**
    *   Ensure you ran `colcon build`. If you recently modified `setup.cfg` or `setup.py`, delete the `build/` and `install/` directories and rebuild.
*   **"No such file or directory: .../culane_res34.pth"**
    *   Ensure you ran the `gdown` command in the Setup step and that the `UFLDV2_DIR` environment variable is exported correctly.
*   **"can't read "Qu(Sensor.Inertial.Vhcl.IN00.Pos_0.x)""**
    *   This occurs if the CarMaker TestRun does not have an Inertial Sensor (IN00) configured, or if the simulation hasn't fully initialized the Data Dictionary before the ROS node connects. Let the simulation start moving before launching.