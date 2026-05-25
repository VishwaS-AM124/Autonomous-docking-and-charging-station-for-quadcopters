#  Autonomous Docking & Charging Station for Quadcopters

> A fully automated, vision-guided landing and charging system for quadcopters — eliminating manual intervention through real-time ArUco marker-based pose estimation, embedded control logic, and wireless sensor communication.

---

##  Overview

This project presents an end-to-end autonomous docking solution for quadcopters, enabling a drone to detect, align with, and land precisely on a charging pad — all without human input. The system combines computer vision, embedded systems programming, and wireless communication to achieve a reliable, repeatable docking workflow suitable for deployment in GPS-denied or indoor environments.

The key challenge addressed is the **last-meter problem** in drone autonomy: once a drone is in the general vicinity of a charging station, it must perform a precise, controlled descent onto a small target. This system solves that with sub-centimeter visual feedback and closed-loop control.

---

##  Key Features

- **ArUco Marker-Based Visual Pose Estimation** — Detects a fiducial marker on the docking pad using OpenCV and computes real-time 6-DoF pose (position + orientation) of the drone relative to the target.
- **Automated Marker-Based Docking** — Closed-loop control pipeline reduces manual intervention by **90%**, guiding the drone autonomously through the full descent sequence.
- **IP-Independent IMU Data Streaming** — Uses **mDNS (Multicast DNS)** over Wi-Fi so the ground station discovers the drone without hardcoded IPs — improving reliability across networks and test environments.
- **Embedded Control & I2C Sensor Integration** — Flight alignment and stabilization are handled via embedded C++ firmware, interfacing with IMU and other sensors over the I2C bus for low-latency feedback.
- **Charging Station Integration** — Upon successful docking, the station initiates charging automatically, completing the full autonomous loop from flight to charge.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        GROUND SYSTEM                        │
│                                                             │
│   ┌─────────────┐     ┌──────────────┐     ┌────────────┐  │
│   │  Camera     │────▶│ Pose         │────▶│ Control    │  │
│   │  (OpenCV)   │     │ Estimation   │     │ Command    │  │
│   └─────────────┘     │ (ArUco)      │     │ Generator  │  │
│                       └──────────────┘     └─────┬──────┘  │
└─────────────────────────────────────────────────┼──────────┘
                                                   │ Wi-Fi (mDNS)
┌─────────────────────────────────────────────────┼──────────┐
│                      DRONE SYSTEM                │          │
│                                                  ▼          │
│   ┌──────────────┐     ┌──────────────┐   ┌──────────────┐ │
│   │  IMU         │────▶│  Embedded    │◀──│  Command     │ │
│   │  (I2C Bus)   │     │  Controller  │   │  Receiver    │ │
│   └──────────────┘     │  (C++)       │   └──────────────┘ │
│                        └──────┬───────┘                    │
│                               │                            │
│                        ┌──────▼───────┐                    │
│                        │  Flight      │                    │
│                        │  Motors /    │                    │
│                        │  ESCs        │                    │
│                        └──────────────┘                    │
└────────────────────────────────────────────────────────────┘
                               │
                        ┌──────▼───────┐
                        │  Charging    │
                        │  Station Pad │
                        │  (ArUco Tag) │
                        └──────────────┘
```

---

## Tech Stack

| Layer | Technology |
|-------|------------|
| Vision & Pose Estimation | Python, OpenCV, ArUco markers |
| Embedded Firmware | Embedded C++ |
| Sensor Communication | I2C protocol (IMU, peripherals) |
| Wireless Communication | Wi-Fi with mDNS (Zeroconf) |
| Control Logic | Custom PID-based flight alignment |

---

## Repository Structure

```
autonomous-drone-docking/
│
├── vision/
│   ├── aruco_detector.py        # ArUco marker detection pipeline
│   ├── pose_estimator.py        # 6-DoF pose computation from marker
│   └── camera_calibration/      # Camera intrinsics & calibration data
│
├── control/
│   ├── docking_controller.py    # High-level docking state machine
│   └── pid_controller.py        # PID control for position alignment
│
├── comms/
│   ├── mdns_discovery.py        # mDNS service discovery (no hardcoded IP)
│   └── telemetry_stream.py      # IMU data streaming over Wi-Fi
│
├── firmware/
│   ├── main.cpp                 # Embedded entry point
│   ├── imu_interface.cpp        # I2C IMU read/write routines
│   ├── motor_control.cpp        # ESC/motor command handler
│   └── command_receiver.cpp     # Parses incoming docking commands
│
├── config/
│   └── params.yaml              # Tunable parameters (PID gains, thresholds)
│
├── docs/
│   └── system_diagram.png       # Architecture diagram
│
└── README.md
```

---

##  How It Works

### 1. Marker Detection & Pose Estimation
A downward-facing camera on the drone (or an upward-facing camera on the station) captures frames continuously. OpenCV's ArUco module detects the marker on the docking pad and computes its **translation and rotation vectors** relative to the camera frame. This gives real-time X, Y, Z offset and yaw error.

### 2. Docking State Machine
The docking controller runs a state machine with the following stages:

```
IDLE → SEARCHING → APPROACHING → ALIGNING → DESCENDING → DOCKED → CHARGING
```

At each stage, the pose estimate feeds into PID controllers that generate velocity setpoints sent to the drone firmware.

### 3. IMU Streaming via mDNS
The drone broadcasts its IMU data (roll, pitch, yaw, accelerations) over Wi-Fi. Instead of requiring a static IP, it registers a **mDNS service** (`_imu._udp.local`), allowing the ground station to discover and connect automatically — making multi-environment testing seamless.

### 4. Embedded Control (Firmware)
The onboard microcontroller handles:
- Reading IMU data via **I2C** at high frequency
- Receiving docking correction commands over Wi-Fi
- Executing fine motor adjustments for precise alignment
- Triggering charging pad activation upon contact confirmation

---

##  Performance Highlights

| Metric | Result |
|--------|--------|
| Manual intervention reduction | **90%** |
| Pose estimation update rate | Real-time (~30 fps) |
| Network discovery method | mDNS (IP-independent) |
| Sensor interface | I2C (low-latency) |
| Environment support | Indoor / GPS-denied |

---

##  Getting Started

### Prerequisites

- Python 3.8+
- OpenCV with ArUco (`opencv-contrib-python`)
- A microcontroller with Wi-Fi (e.g., ESP32) flashed with the firmware
- IMU sensor connected via I2C (e.g., MPU-6050)

### Installation

```bash
git clone https://github.com/yourusername/autonomous-drone-docking.git
cd autonomous-drone-docking

pip install -r requirements.txt
```

### Running the Vision & Control Pipeline

```bash
# Start the docking controller (connects via mDNS automatically)
python control/docking_controller.py

# Or run pose estimation standalone for testing
python vision/pose_estimator.py --camera 0 --marker-id 42
```

### Flashing Firmware

```bash
# Using PlatformIO
cd firmware/
pio run --target upload
```

---

##  ArUco Marker Setup

Print the ArUco marker (Dictionary: `DICT_4X4_50`, Marker ID: `42`) at the recommended size and place it flat on the center of the docking pad. Ensure consistent lighting for reliable detection.

The marker size used for calibration is defined in `config/params.yaml`:

```yaml
aruco:
  marker_id: 42
  marker_size_meters: 0.15
  dictionary: DICT_4X4_50
```

---

##  Configuration

Key tunable parameters in `config/params.yaml`:

```yaml
pid:
  lateral_kp: 0.4
  lateral_ki: 0.01
  lateral_kd: 0.1
  descent_kp: 0.3

docking:
  alignment_threshold_cm: 2.0
  descent_speed_ms: 0.1
  final_descent_altitude_cm: 10.0

mdns:
  service_name: "_imu._udp.local"
  port: 5005
```

---

##  Notes

- This repository contains only the open-source components of the system. Proprietary integration details and deployment-specific configurations are not included.
- The firmware is written for an **ESP32-class microcontroller** but can be adapted to other platforms with I2C and Wi-Fi support.
- Camera calibration files must be generated for your specific camera before use. See `vision/camera_calibration/README.md`.

---

##  Skills Demonstrated

- **Computer Vision** — Real-time fiducial marker detection and 6-DoF pose estimation
- **Embedded Systems** — C++ firmware, I2C protocol, motor/ESC control
- **Networked Systems** — mDNS-based service discovery, UDP telemetry streaming
- **Control Systems** — PID-based closed-loop drone alignment
- **System Integration** — Bridging vision, networking, and embedded layers end-to-end

---

##  License

This project is for portfolio demonstration purposes. All rights reserved.

---


