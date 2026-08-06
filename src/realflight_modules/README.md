# 实机飞行模块

本目录包含 LIO、PX4 通信与控制相关模块。
Firefly ROC-RK3588S-PC 的系统资料见[官方文档](https://wiki.t-firefly.com/zh_CN/ROC-RK3588S-PC/index.html)。

## 1. 拉取源码

首次克隆仓库时建议同时初始化 submodule：

```bash
git clone -b dev_nanobot --recurse-submodules https://github.com/zhan994/Diff-Planner.git
```

仓库已存在时，使用以下命令拉取 FAST_LIO：

```bash
git submodule update --init --recursive
```

按使用的雷达安装对应驱动：

```bash
# RoboSense
git clone https://github.com/zhan994/nanobot_sdk.git \
  src/realflight_modules/nanobot_sdk
```

## 2. 安装依赖

以下命令适用于 ROS Noetic：

```bash
sudo apt update
sudo apt install -y \
  build-essential cmake git pkg-config \
  libapr1-dev libaprutil1-dev libboost-all-dev \
  libeigen3-dev libfmt-dev libgoogle-glog-dev \
  libompl-dev ompl-demos libpcap-dev libpcl-dev \
  ros-noetic-pcl-conversions ros-noetic-pcl-ros \
  ros-noetic-rosfmt
```

安装 MAVROS 及 GeographicLib 数据集：

```bash
sudo apt install -y ros-noetic-mavros ros-noetic-mavros-extras
sudo /opt/ros/noetic/lib/mavros/install_geographiclib_datasets.sh
```

使用 Livox 雷达时，还需安装Livox驱动：

```bash
# Livox SDK2
git clone https://github.com/Livox-SDK/Livox-SDK2.git
cd Livox-SDK2
mkdir build && cd build
cmake ..
make -j
sudo make install

# Livox
git clone https://github.com/Livox-SDK/livox_ros_driver2.git \
  src/realflight_modules/livox_ros_driver2
```

## 3. 配置 RoboSense 雷达

在 `rslidar_sdk/config/config.yaml` 中设置雷达型号和 IMU 端口：

```yaml
lidar_type: RSAIRY
imu_port: 6688
```

在 `rslidar_sdk/CMakeLists.txt` 中开启 IMU 数据解析，并将点云类型设为 `XYZIRT`：

```cmake
set(ENABLE_IMU_DATA_PARSE ON)
set(POINT_TYPE XYZIRT)
```

### 配置静态 IP

先确认系统已安装 NetworkManager：

```bash
command -v nmcli
```

若命令有输出，创建并启用雷达网口配置：

```bash
sudo nmcli connection add \
  type ethernet \
  ifname eth0 \
  con-name lidar-static \
  ipv4.method manual \
  ipv4.addresses 192.168.1.102/24 \
  ipv4.never-default yes \
  ipv6.method disabled \
  connection.autoconnect yes

sudo nmcli connection up lidar-static
```

检查网口地址和路由：

```bash
ip -br addr
ip route
```

正常输出示例：

```text
eth0   UP   192.168.1.102/24 (雷达IP)
wlan0  UP   192.168.8.238/24 (本机IP)
```

## 4. 编译

在仓库根目录执行：

```bash
catkin_make
```

## 5. 配置 PX4 EKF2 (LIO->PX4)

`px4bridge` 仅发送位置和姿态，建议使用以下参数：

| 参数 | 建议值 | 说明 |
| --- | ---: | --- |
| `EKF2_EV_CTRL` | `11` | 融合视觉水平位置、垂直位置和偏航角，不融合速度 |
| `EKF2_EV_NOISE_MD` | `1` | 使用 PX4 参数中的视觉噪声，不使用消息协方差 |
| `EKF2_EVP_NOISE` | `0.10` | 视觉位置噪声初始值，单位为 m |
| `EKF2_EVA_NOISE` | `0.05` | 视觉姿态噪声初始值，单位为 rad |
| `EKF2_EV_DELAY` | 实测值 | 视觉数据相对 IMU 的固定延迟，单位为 ms |
| `EKF2_EV_POS_X/Y/Z` | `0` | 输入为机体位姿时设为 `0` |

按实际场景补充配置：

- 使用 LIO 的 Z 轴作为高度基准时，将 `EKF2_HGT_REF` 设为 `3`。
- 室内完全不使用 GNSS 时，可将 `EKF2_GPS_CTRL` 设为 `0`。
- 修改 EKF 参数后重启飞控。

### 标定里程计延迟

先在 QGroundControl 中配置日志：

| 参数 | 设置 |
| --- | --- |
| `SDLOG_PROFILE` | 勾选 bit 7：`Computer Vision and Avoidance` |
| `SDLOG_MODE` | 台架测试时设为 `2`：`From boot until shutdown` |

安装分析脚本依赖并绘制 ULog 中的视觉与 IMU 偏航角速度：

```bash
python3 -m pip install pyulog numpy matplotlib
python3 src/realflight_modules/px4bridge/scripts/plot_ev_delay.py <log.ulg>
```

根据两条曲线的固定时间差设置 `EKF2_EV_DELAY`。该参数只能补偿固定延迟；标定结束后可将 `SDLOG_MODE` 恢复为 `0`，以减少日志量。
