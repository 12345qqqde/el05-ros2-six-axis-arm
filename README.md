# EL05 六轴机械臂 · ROS 2 工程

EL05（RobStride 准直驱电机）六轴机械臂的完整工程：SolidWorks 导出的 URDF/RViz 描述、正逆运动学与工作空间分析、真机 CAN 伺服、标定工具，以及接到上位机（hand-host）的集成包。

> **代码来源**：`ros2_arm_ws/` 与 `el05_serial_ws/` 的实现由 **Raki（秦均灿）** 编写（3433253304@qq.com），原始仓库 [Rakiv-dsh/arm-urdf](https://github.com/Rakiv-dsh/arm-urdf)。2026-11-06 按 2026-11-02 版本并入本仓库，仅做目录改名，未改源码。详见 [CREDITS.md](CREDITS.md)。

- 机械结构：6 个关节，其中 joint1~joint5 有电机（CAN ID 21~25），joint6 无电机（从动，只做运动学占位）
- 通信：SocketCAN（USB-CANHUB / candleLight，1 Mbps，EL05 私有协议，CAN 扩展帧）
- ROS 版本：ROS 2 Jazzy（Ubuntu 24.04 / WSL2）

## 目录结构

| 路径 | 内容 |
|---|---|
| `ros2_arm_ws/src/arm_urdf/` | 机械臂描述包：URDF（含 SolidWorks 网格）、RViz 配置、`joint_limits.yaml`（**关节限位与电机映射的唯一数据源**）、滑块联调与标定工具 |
| `el05_serial_ws/src/robstride_ros_sample/` | RobStride 电机 ROS 2 驱动与示例（`rs_motor_ros2`：CAN 收发、使能/运控/反馈、正弦测试、限位搜索、状态发布） |
| `el05_serial_ws/src/el05_serial_tools/` | 串口监视等辅助工具包 |
| `integrations/arm_el05/` | 上位机手型包（`dexhand.hand.v1`），把机械臂当成 5 关节“手”接入 hand-host |
| `tools/` | 环境与硬件排障脚本：CAN 起网、CAN 抓包、电机只读探测、改电机 ID、虚拟 EL05 电机（vcan） |
| `docs/` | 正逆运动学分析报告与工作空间图 |

## 快速开始

### 1. 编译

```bash
# 机械臂描述侧
cd ros2_arm_ws && colcon build && source install/setup.bash

# 电机驱动侧（含 rs_motor_ros2）
cd el05_serial_ws && colcon build && source install/setup.bash
```

依赖：`ros-jazzy-robot-state-publisher` `ros-jazzy-rviz2` `ros-jazzy-joint-state-publisher-gui` `ros-jazzy-can-msgs`、`can-utils`。Ubuntu 侧 CAN 起网见 `el05_serial_ws/src/robstride_ros_sample/README.md`。

### 2. 只看模型（不接硬件）

```bash
ros2 launch arm_urdf display.launch.py
```

### 3. 滑块联调（可同时驱动真机）

```bash
# 只仿真
ros2 launch arm_urdf slider_servo.launch.py servo:=false
# 真机：先起 can0，再启动；默认还要 /arm/enable=true 才真动
sudo bash tools/setup_can0_wsl.sh can0
ros2 launch arm_urdf slider_servo.launch.py can_interface:=can0
```

常用参数：`max_speed`（目标斜坡 rad/s）、`limit_cur`（使能前写入的电流限幅）、`watchdog_ms`（多久收不到 `/joint_states` 就失能）、`active_joints`（只驱动指定关节，调方向时用）。

### 4. 没有硬件时验证整条链路

```bash
sudo bash tools/demo_virtual.sh 8     # vcan + 虚拟 EL05 ×5 + 控制器 → /arm/joint_states
```

## 关节、电机与限位

唯一数据源是 `ros2_arm_ws/src/arm_urdf/config/joint_limits.yaml`。改完执行
`python3 ros2_arm_ws/src/arm_urdf/tools/sync_limits.py`，会同步刷新 URDF 的 `<limit>` 与控制器 `five_dof_controller.yaml`（限位/方向/偏置/电机号）。

| 关节 | 轴 | 电机 CAN ID | 当前限位 |
|---|---|---|---|
| joint1 | 竖直（底座回转） | 21 | ±180° |
| joint2 | X（肩俯仰） | 22 | −23° ~ +50° |
| joint3 | 竖直（肘回转） | 23 | ±180° |
| joint4 | Y（腕1） | 24 | 0° ~ +16° |
| joint5 | 竖直（腕2） | 25 | ±180° |
| joint6 | Y（腕3） | 无 | ±90° |

映射关系：`mechPos = offset + dir * gear * joint_angle`。零位已烘进电机（`0x7019` 读 0），因此 `offset` 即零位读数。

## 运动学与工作空间

`docs/正逆运动学.md` 是完整分析（只算数学，不碰真机），要点：

- 满行程下末端可达范围约 **1.6 cm ~ 34.4 cm**，Z 从 −17 cm 到 +34 cm；
- **当前 joint2、joint4 被限死在很小的角度里**，可用工作空间退化成 Z 0.198~0.341 m 的薄盘，这是“拖滑块感觉动不了”的根本原因；
- 逆解用阻尼最小二乘（多初值重试 + 限位夹取），40 个随机可达位姿 100% 解出，位置误差中位 0.1 mm；
- 奇异位形比例：满行程 48.7%（条件数 > 200），当前限位 99.0% —— 放开 joint2/joint4 后显著改善。

重算：`python3 ros2_arm_ws/src/arm_urdf/tools/kinematics.py`（输出到 `arm_urdf/docs/`）。

## 标定与工具

| 脚本 | 用途 |
|---|---|
| `tools/el05_probe.py` | 只读探测 CAN 上哪些电机在线（不发使能帧） |
| `tools/el05_set_id.py` | 修改电机 CAN ID |
| `tools/can_sniff.py` | CAN 抓包与协议解码 |
| `tools/el05_vcan_sim.py` | 虚拟 EL05 电机，跑在 SocketCAN 上，无需硬件 |
| `tools/bringup_can.sh` / `setup_can0_wsl.sh` | WSL 内拉起 CAN 接口并逐路探测 |
| `arm_urdf/tools/calibrate.py` `record_range.py` `set_zero_here.py` `zero_*.py` | 零位写入、行程记录、限位标定 |

## 上位机集成

`integrations/arm_el05/` 是一个 `dexhand.hand.v1` 手型包：把机械臂当 5 关节“手”接进 hand-host（端点 `can0`、协议 `socketcan`、1 Mbps）。字段对照与迁移步骤见 `integrations/arm_el05/docs/迁移说明.md`；`sync_from_ros.py` 可从 `joint_limits.yaml` 刷新 `manifest.json` / `calibration.json`，保证 ROS 侧与上位机侧不打架。

## 已知限制

- joint6 无电机，逆解出的第 6 轴角度目前无法执行。
- joint2 / joint4 限位远小于机构能力，工作空间与可操作性受限（见上）。
- 零位与限位是逐台标定出来的，换臂或动过机械零位需要重新标定。
- 真机运动前必须确认电流限幅（默认 6 A）与看门狗设置。

## 致谢与授权

- `el05_serial_ws/src/robstride_ros_sample` 基于 RobStride 官方示例（原作者 DR.MuShibo）修改。
- URDF 由 SolidWorks to URDF Exporter 生成，网格文件为 EL05 机械臂模型。
- 各包声明的许可证见对应 `package.xml`（`arm_urdf` 为 MIT）。
