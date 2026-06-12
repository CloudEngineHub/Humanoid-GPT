# Onboard Deployment Guide for Unitree G1

This guide walks through deploying **Humanoid-GPT** directly on the Unitree G1's
onboard Jetson computer.  All inference (walk policy, tracking policy, motion
retargeting) runs on-device — no external workstation required.

The entire workflow happens over SSH from your laptop.

## Architecture Overview

```
┌──────────────────────────────────────────────────────┐
│                   Unitree G1 Robot                   │
│                                                      │
│  ┌───────────────────────────────────────────────┐   │
│  │        Jetson Orin (Onboard Computer)         │   │
│  │                                               │   │
│  │  ┌──────────────┐   ┌──────────────────────┐  │   │
│  │  │ Walk Policy  │   │ Tracking Policy      │  │   │
│  │  │ (ONNX / TRT) │   │ (ONNX / TRT)         │  │   │
│  │  └──────┬───────┘   └─────────┬────────────┘  │   │
│  │         │                     │               │   │
│  │         ▼                     ▼               │   │
│  │  ┌──────────────────────────────────────────┐ │   │
│  │  │        play_track_onboard.py             │ │   │
│  │  │   (Curses Terminal UI + Control Loop)    │ │   │
│  │  └──────────────┬───────────────────────────┘ │   │
│  │                 │                             │   │
│  │      ┌──────────┴──────────┐                  │   │
│  │      ▼                     ▼                  │   │
│  │  ┌────────┐         ┌──────────────┐          │   │
│  │  │  DDS   │ (eth0)  │  Noitom WiFi │ (wlan0)  │   │
│  │  │ Motor  │         │  Retarget    │          │   │
│  │  │ Control│         │  Subprocess  │          │   │
│  │  └───┬────┘         └──────┬───────┘          │   │
│  └──────┼─────────────────────┼──────────────────┘   │
│         ▼                     │                      │
│   Motor Controllers           │                      │
│   (29 DOF joints)             │                      │
└───────────────────────────────┼──────────────────────┘
                                │ WiFi
                    ┌───────────┴─────────────┐
                    │  Noitom Perception Neuron │
                    │  (PNLink Streaming)       │
                    └──────────────────────────┘
```

**No Ethernet cable needed during tracking.**  Once the environment is set up
on the G1, the entire tracking workflow is cable-free:

- **Motor control** uses the G1's **internal bus** (`eth0` inside the robot,
between Jetson and the motor controller board) — no external cable.
- **SSH access** to the G1 goes over **WiFi** (`wlan0`).
- **Noitom PNLink** streams mocap data over the same **WiFi** network.

## Step 1: Robot Bring-Up

### 1.1 Power On

1. Short press the battery button, then long press (~2 seconds) to power on.
2. Wait for the head indicator to stabilize (~30 seconds).

### 1.2 Enter Debug Mode

Press `L2 + R2` on the physical remote controller to enter debug (low-level)
mode.  This is required for direct motor control.

## Step 2: SSH into the Unitree G1

### 2.1 Find the G1's IP Address

The G1's onboard Jetson typically has a fixed IP address.  Common defaults:

| Connection              | IP                 | Notes                   |
|-------------------------|--------------------|-------------------------|
| Ethernet (direct cable) | `192.168.123.164`  | Always available        |
| WiFi                    | Assigned by router | Check router DHCP table |

If you connect your laptop directly to the G1 via Ethernet, configure your
laptop's Ethernet interface to a static IP in the same subnet, such as 192.168.123.100 with netmask 255.255.255.0

### 2.2 Connect via SSH

```bash
ssh unitree@192.168.123.164
# Default password: 123
```

Upon login you will be prompted to select a ROS distribution:

```
ros:foxy(1) noetic(2)
```
**Select `1` (foxy).**  The Unitree SDK2 used by this project is built on
ROS 2 / DDS, which corresponds to the Foxy distribution.

## Step 3: Set Up WiFi

Connect Wifi using Network Manager or wpa_supplicant.
You can using some agent tools (such as Claude Code or Cursor) to help you connect Wifi.

After connection, your laptop can find the robot by its new WiFi IP:

```bash
# From your laptop, find the robot on the same network:
ping <robot_wifi_ip>
ssh unitree@<robot_wifi_ip>
```

## Step 4: Install Humanoid-GPT Environment

### 4.1 Install Miniconda or UV in the robot

```bash
# Download Miniconda for aarch64
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-aarch64.sh
bash Miniconda3-latest-Linux-aarch64.sh -b -p $HOME/miniconda3
eval "$($HOME/miniconda3/bin/conda shell.bash hook)"
conda init bash
source ~/.bashrc
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/main
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/r
```

### 4.2 Clone the Repository

```bash
cd ~
git clone https://github.com/GalaxyGeneralRobotics/Humanoid-GPT.git
cd Humanoid-GPT
```

### 4.3 Create Conda Environment

```bash
conda create -n h-gpt python=3.12 -y
conda activate h-gpt
cd ~/Humanoid-GPT
pip install -e .
```

## 已知问题：ONNX Runtime Provider 警告

Jetson Orin NX + JetPack 5 (L4T R35.x) 上，pip 安装的  仅支持 CPU
（Python 3.12 + aarch64 没有可用的 CUDA/TensorRT provider wheel）。代码中
 和  现在会先调用
 检查实际可用的 provider，仅请求存在的 provider，
消除以下警告：



性能不受影响——小型 MLP 策略在 CPU 上可达 ~1000+ Hz，远超 50 Hz 控制频率要求。

未来如果有适配该平台的 onnxruntime-gpu wheel（如升级 JetPack 6 或社区编译），
代码会自动启用 CUDA/TensorRT provider，无需改动。

### 修改的文件

-  —  和  增加
   检查。
-  —  调用处同样增加 provider 可用性检查。

## 已知问题：MuJoCo Warp 导入警告

MuJoCo 的  子模块在未安装 NVIDIA Warp 时会直接  到 stdout：



由于是  而非 ，无法通过 warning filter 抑制。修复方式
是在  时临时用一个过滤器包装 stdout，仅过滤包含上述特定内容的输出，
其他 print 正常通过。

### 修改的文件

-  — 在  前后用
   包装 stdout，过滤 warp 相关的无用输出。

## Step 5: Install Third-Party Libraries

### 5.1 Download and Extract

```bash
cd ~/Humanoid-GPT
pip install gdown
gdown https://drive.google.com/uc?id=1bfgFhrv6tfuDOkt11AOJAO2IHTRXlYey -O thirdparty.zip
unzip thirdparty.zip
rm thirdparty.zip
```

After extraction:

```
thirdparty/
├── GMR-galbot/              # Motion retargeting
├── noitom/                  # PNLink mocap client
├── cyclonedds/              # DDS middleware (C library)
└── unitree_sdk2_python/     # Unitree motor control SDK
```

### 5.2 Build CycloneDDS

```bash
cd ~/Humanoid-GPT/thirdparty/cyclonedds
mkdir -p build install
cd build
cmake .. -DCMAKE_INSTALL_PREFIX=../install
cmake --build . --target install -j$(nproc)
cd ~/Humanoid-GPT
```

> **Note**: You need `cmake` and a C compiler. On Jetson:
>
> ```bash
> sudo apt-get update && sudo apt-get install -y cmake build-essential
> ```

### 5.3 Install Unitree SDK

```bash
export CYCLONEDDS_HOME="$HOME/Humanoid-GPT/thirdparty/cyclonedds/install"
pip install -e thirdparty/unitree_sdk2_python
```

Add the environment variable to your shell profile so it persists:

```bash
echo 'export CYCLONEDDS_HOME="$HOME/Humanoid-GPT/thirdparty/cyclonedds/install"' >> ~/.bashrc
source ~/.bashrc
```

### 5.4 Install Retargeting Libraries

```bash
pip install -e thirdparty/GMR-galbot
pip install -e thirdparty/noitom          # Only needed for Noitom PNLink
```

## Step 6: Control Interface

### Jetson runtime requirement

Before every real-robot run on the G1 Jetson, lock the Jetson clocks:

```bash
sudo jetson_clocks
```

Alternatively, flash the robot computer to **JetPack 6.2** and deploy with the TensorRT (`--use-trt`) runtime.

### Tracking Command

```bash
python -m deploy.onboard_deploy.play_track_onboard --onnx_track onnx-path
```

### Terminal UI

The curses-based interface displays mode, velocity bars, and status directly in
the SSH terminal:

```
     Humanoid-GPT Onboard Deploy
MODE  [0:Walk]  1:Online  2:Trk0
VELOCITY
  X (W/S)  [..........│######....] +0.30
  Y (A/D)  [..........│..........] +0.00
  Yaw(Q/E) [..........│..........] +0.00

STATUS
  Robot: Running     Mocap: Connected (pnlink)
  Freq:  49.8 Hz    Steps: 4521

[0-9]Mode [WASDQE]Vel [R]Reset [Space]EStop [Esc]Quit
Log: /tmp/humanoid_gpt_onboard/deploy_20260408_143022.log
```

### Keyboard Controls


| Key       | Function                             |
|-----------|--------------------------------------|
| `0`       | Walk mode (velocity control)         |
| `1`       | Online retarget mode (Noitom)        |
| `2`-`9`   | Offline trajectory tracking          |
| `W` / `S` | Linear velocity X (+/-)              |
| `A` / `D` | Linear velocity Y (+/-)              |
| `Q` / `E` | Yaw angular velocity (+/-)           |
| `R`       | Reset tracking state                 |
| `Space`   | **Emergency stop** (goes to damping) |
| `Esc`     | Quit                                 |


Velocity damping is automatic — release the key and velocity decays to zero.

### Physical Remote Controller

The startup sequence uses the wireless remote for safety:


| Button   | Function                            |
|----------|-------------------------------------|
| `start`  | Confirm damping → stand up          |
| `A`      | Confirm standing → enter locomotion |
| `select` | **Emergency stop** at any time      |


## CLI Arguments


| Argument         | Default                             | Description                           |
|------------------|-------------------------------------|---------------------------------------|
| `--net`          | `eth0`                              | DDS network interface (motor bus)     |
| `--freq`         | `50`                                | Control loop frequency (Hz)           |
| `--debug`        | `False`                             | Dry run — no motor commands published |
| `--use-trt`      | `True`                              | Use TensorRT for inference            |
| `--onnx-walk`    | `storage/ckpts/G1-Walk/...onnx`     | Walk policy path                      |
| `--onnx-track`   | `storage/ckpts/pns_wo_priv216.onnx` | Tracking policy path                  |
| `--track-dir`    | `storage/test`                   | Offline trajectory folder             |
| `--no-mocap`     | `False`                             | Disable online motion capture         |
| `--mocap-type`   | `pnlink`                            | `pnlink` or `xsens`                   |
| `--human-height` | `1.7`                               | Retarget height calibration (meters)  |
| `--buffer-ms`    | `30.0`                              | Jitter buffer for mocap stream (ms)   |
| `--enable-hand`  | `False`                             | Enable Dex3-1 hand control            |


## Logs

All runtime logs are written to `/tmp/humanoid_gpt_onboard/` since curses
owns the terminal.  Check logs for debugging:

## Quick Reference Card

```bash
# First-time setup (on G1 Jetson, via SSH)
conda activate h-gpt
cd ~/Humanoid-GPT

# Required before each real-robot run on Jetson
sudo jetson_clocks

# Tracking
python -m deploy.onboard_deploy.play_track_onboard --onnx_track onnx-path
```
