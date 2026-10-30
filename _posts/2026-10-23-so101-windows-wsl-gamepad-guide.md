---
layout: post
title: SO101：Windows、WSL 与蓝牙手柄实机控制手册
date: 2026-10-23 00:00:00 +0800
description: 汇总 SO101 从臂在 Windows 与 WSL2 下的串口转发、环境检查、校准、蓝牙手柄控制和故障排查流程。
tags: [LeRobot, SO101, WSL, 手柄控制]
categories: [机器人开发]
published: true
toc:
  sidebar: left
---

更新日期：2026-10-23。

适用对象：当前这台 Windows 电脑、WSL2 的 Ubuntu-24.04、已安装的 `so101-cartesian` Conda 环境，以及工程中已经加入的浏览器手柄桥接功能。

本文覆盖本次配置中实际遇到的问题和日常使用的常见分支。故障处理章节按现象选择执行，不要把所有修复命令从头运行一遍。

## 目录

1. [日常启动速查](#1-日常启动速查)
2. [本机配置与终端说明](#2-本机配置与终端说明)
3. [Windows：将从臂串口转发到 WSL](#3-windows将从臂串口转发到-wsl)
4. [Ubuntu：检查串口、权限和占用](#4-ubuntu检查串口权限和占用)
5. [检查 Conda 环境和工程文件](#5-检查-conda-环境和工程文件)
6. [从臂通信检查与重新校准](#6-从臂通信检查与重新校准)
7. [Windows：蓝牙连接手柄](#7-windows蓝牙连接手柄)
8. [启动服务和浏览器控制](#8-启动服务和浏览器控制)
9. [停止、退出、断电和恢复](#9-停止退出断电和恢复)
10. [常见故障排查](#10-常见故障排查)
11. [日志、检查与备份](#11-日志检查与备份)
12. [环境损坏或迁移后的恢复](#12-环境损坏或迁移后的恢复)
13. [文件与验证记录](#13-文件与验证记录)

## 1. 日常启动速查

适用于环境已经安装、从臂校准仍有效的情况。首次配置或遇到错误时，按后面的详细章节排查。

### 第一步：固定机械臂并连接控制板

固定从臂底座，整理运动范围内的线缆和物品。连接控制板 USB；需要读取电机或执行动作时，按硬件要求接通从臂电机电源。USB 枚举成功不代表电机已经供电。

### 第二步：打开 Ubuntu-24.04

**执行位置：Windows PowerShell，普通权限。**

```powershell
wsl -d Ubuntu-24.04 -u oong7
```

保持这个 Ubuntu 终端开着，供后面使用。

### 第三步：转发机械臂控制板

**执行位置：另开一个 Windows 管理员 PowerShell。**

```powershell
usbipd list
```

找到从臂控制板 `USB-Enhanced-SERIAL CH343`。本机记录是 `VID:PID = 1a86:55d3`，Windows 曾显示为 COM3。每次都核对实际 BUSID，不要只凭旧编号判断。

确认这次仍为 `2-3` 后执行；若编号改变，先修改变量：

```powershell
$armBusId = '2-3'
usbipd bind --busid $armBusId
usbipd attach --wsl --busid $armBusId
usbipd list
```

期望状态：对应控制板显示 `Attached`。如果已经是 `Attached`，直接继续，不必重复连接；如果已经是 `Shared`，只需执行 `attach`。

### 第四步：确认 Ubuntu 串口可用

**执行位置：刚才打开的 Ubuntu 终端。**

```bash
ls -l /dev/serial/by-id/
ls -l /dev/ttyACM*
```

本机从臂稳定路径应为：

```text
/dev/serial/by-id/usb-1a86_USB_Single_Serial_5AB0181922-if00
```

### 第五步：手柄连接 Windows 蓝牙

在 Windows 设置中把手柄连接到蓝牙。手柄留在 Windows，不转发手柄接收器或蓝牙适配器到 WSL。

### 第六步：启动真实从臂服务

**执行位置：Ubuntu 终端。**

```bash
bash ~/Lets-Build-Robots-S1-SO101-main/start-cartesian-gamepad.sh --real
```

保持终端运行。若提示端口被占用，先检查是否已有服务，不要重复启动。

### 第七步：打开 Windows 浏览器

使用 Windows Chrome，打开：

`http://127.0.0.1:4604`

按以下顺序操作：

1. 保持页面在前台，按一下手柄 A 键后松开。
2. 摇杆回中、扳机和按钮全部松开，点击上方“连接手柄输入”。
3. 等待手柄名称出现，接收帧数递增。
4. 确认下方 `Follower = Real arm`、`Mode = Gamepad`，点击 `Enable Gamepad`。
5. 准备好观察机械臂后，点击 `Hold and follow`，或按手柄 Start。
6. 首次轻推摇杆，确认方向后再扩大操作幅度。

**`Stop` / Back 只停止跟随，仍保持力矩。托住机械臂后，点击 `Release torque` 才释放力矩。**

## 2. 本机配置与终端说明

### 2.1 控制链路

```text
飞智手柄 ──蓝牙──> Windows Chrome ──本机 HTTP──> WSL Cartesian 控制循环
                                                     │
SO101 从臂 <──电机总线── 控制板 <──USB/IP 串口─────────────┘
```

两条连接各自独立：

| 设备                | 连接与使用位置                | 是否需要 usbipd |
| ------------------- | ----------------------------- | --------------- |
| Follower 从臂控制板 | Windows USB → WSL 串口        | 需要            |
| 飞智手柄            | Windows 蓝牙 → Windows 浏览器 | 不需要          |
| Windows 蓝牙适配器  | 保留给 Windows                | 不需要          |
| Leader 主臂         | 本流程不使用                  | 不需要          |
| 摄像头              | 本流程不使用                  | 不需要          |

本流程不需要在 WSL 中安装手柄 `xpad` 驱动，不需要把手柄变成 `/dev/input/js0`，也不需要摄像头、Isaac Sim 或 Isaac Lab。

### 2.2 本机固定配置

| 项目               | 本机配置                                                       |
| ------------------ | -------------------------------------------------------------- |
| WSL 发行版         | `Ubuntu-24.04`，WSL2                                           |
| Linux 用户         | `oong7`                                                        |
| 工程根目录         | `/home/oong7/Lets-Build-Robots-S1-SO101-main`                  |
| Conda 环境         | `so101-cartesian`                                              |
| Python             | `/home/oong7/miniconda3/envs/so101-cartesian/bin/python`       |
| LeRobot            | `0.6.1`；本课程脚本有版本检查                                  |
| placo / viser      | `0.9.15` / `1.1.0`                                             |
| 控制板 VID:PID     | `1a86:55d3`                                                    |
| 从臂稳定串口       | `/dev/serial/by-id/usb-1a86_USB_Single_Serial_5AB0181922-if00` |
| 从臂 ID            | `so101-follower`                                               |
| 从臂校准文件       | 工程内 `calibration/follower/so101-follower.json`              |
| 手柄与控制统一入口 | `http://127.0.0.1:4604`                                        |
| 真实从臂内部页面   | `http://127.0.0.1:4603`                                        |
| 模拟从臂内部页面   | `http://127.0.0.1:4602`                                        |

COM 号、BUSID、Linux 的 `/dev/ttyACM0` 编号和运行进程 PID 都可能变化。稳定串口路径在更换控制板后也会变化；更换硬件时必须重新识别。

### 2.3 命令在哪执行

| 终端                      | 如何打开                                          | 执行的内容                                          |
| ------------------------- | ------------------------------------------------- | --------------------------------------------------- |
| Windows 管理员 PowerShell | 开始菜单搜索 PowerShell，右键“以管理员身份运行”   | `usbipd bind`、必要时 `--force`、`unbind`、服务管理 |
| Windows 普通 PowerShell   | 普通打开终端                                      | `wsl`、`usbipd attach`、检查本地网页                |
| Ubuntu Bash               | 在 PowerShell 执行 `wsl -d Ubuntu-24.04 -u oong7` | Linux 串口检查、Conda、校准、启动控制服务           |
| Windows Chrome            | Windows 开始菜单打开 Chrome                       | 配对后的手柄读取、浏览器控制                        |

`sudo` 提升的是 Ubuntu 权限，不会让 Windows PowerShell 变成管理员。Windows 的 UAC 管理员权限与 Ubuntu 的 root 权限是两回事。

## 3. Windows：将从臂串口转发到 WSL

### 3.1 检查 WSL 与 usbipd

**执行位置：Windows PowerShell。**

```powershell
wsl --list --verbose
usbipd --version
```

期望能找到 `Ubuntu-24.04`，且 VERSION 为 `2`。本机核对过的 usbipd-win 版本是 `5.3.0`。

当前机器已经安装这些工具，无需重复安装。如果另一次恢复环境时提示找不到 `usbipd`，可在管理员 PowerShell 安装：

```powershell
winget install --interactive --exact dorssel.usbipd-win
```

按安装界面完成操作；安装器若要求重启，先结束机械臂控制。安装后重新打开 PowerShell，再执行 `usbipd --version`。

### 3.2 先让目标 Ubuntu 运行

```powershell
wsl -d Ubuntu-24.04 -u oong7
```

此命令会进入 Linux Shell。下面的 `usbipd` 命令要在另一个 **Windows** PowerShell 执行，不要误输进这个 Ubuntu Shell。

### 3.3 识别控制板

**执行位置：Windows 管理员 PowerShell。**

```powershell
usbipd list
```

本机曾出现：

```text
BUSID  VID:PID    DEVICE                              STATE
2-3    1a86:55d3  USB-Enhanced-SERIAL CH343 (COM3)     Attached
```

`Persisted` 区域是历史绑定记录，不能把历史 COM4 或手柄记录当成当前插入的设备。应在 `Connected` 区域识别当前控制板。

如果多块相同控制板难以区分，只在机械臂已经停止、已妥善支撑并切断电机电源时，通过拔插目标 USB 前后对比列表来确认。不要在运动中用拔线的方法找端口。

### 3.4 首次绑定

确认 BUSID 后设置变量。新开 PowerShell 窗口后，需要重新设置这个变量：

```powershell
$armBusId = '2-3'
usbipd bind --busid $armBusId
```

`bind` 通常只需对该设备做一次。对应状态已经是 `Shared` 或 `Attached` 时，不必重新绑定。

### 3.5 连接到 WSL

```powershell
usbipd attach --wsl --busid $armBusId
usbipd list
```

对应状态变成 `Attached` 即为 USB/IP 连接成功。

也可以明确指定用 Ubuntu-24.04 发起连接：

```powershell
usbipd attach --wsl Ubuntu-24.04 --busid $armBusId
```

本机版本可能提示：

```text
Selecting a specific distribution is no longer required.
the device will be available in all WSL 2 distributions.
Detected networking mode 'mirrored'.
Using IP address 127.0.0.1 to reach the host.
```

这些 `info` 不是失败。多个 WSL2 发行版共享 WSL 虚拟机的 USB 设备，选择发行版不代表设备被独占隔离。本流程固定在 Ubuntu-24.04 中运行控制程序，不要让另一发行版同时打开同一串口。[微软 USB 转发说明](https://learn.microsoft.com/en-us/windows/wsl/connect-usb)

### 3.6 什么时候需要重新 attach

- 控制板 USB 拔插后。
- Windows 或 WSL 重启后。
- 执行 `wsl --shutdown` 后。
- USB/IP 意外断开后。

每次先运行 `usbipd list`，重新核对 BUSID，再执行 `attach`。不需要因此每次重装驱动或重新校准。

## 4. Ubuntu：检查串口、权限和占用

### 4.1 设置后续命令使用的变量

**执行位置：Ubuntu Bash。**

```bash
export PROJECT_DIR="$HOME/Lets-Build-Robots-S1-SO101-main"
export CARTESIAN_PY="$HOME/miniconda3/envs/so101-cartesian/bin/python"
export FOLLOWER_PORT="/dev/serial/by-id/usb-1a86_USB_Single_Serial_5AB0181922-if00"
cd "$PROJECT_DIR"
```

这些变量只对当前 Shell 及其子进程生效。重新开 Ubuntu 终端后，重新执行本段即可。

### 4.2 检查设备是否出现

```bash
ls -l /dev/serial/by-id/
ls -l /dev/ttyACM*
readlink -f "$FOLLOWER_PORT"
```

本机稳定路径通常指向 `/dev/ttyACM0`。脚本使用 `/dev/serial/by-id/` 路径，避免 ACM 编号改变后误连。

如需查看 USB VID/PID：

```bash
lsusb
```

如果提示 `lsusb: command not found`，这只说明诊断工具没有安装，不等于 USB 转发失败。可安装常用诊断工具：

```bash
sudo apt update
sudo apt install usbutils psmisc
```

其中 `usbutils` 提供 `lsusb`，`psmisc` 提供后面用到的 `fuser`。仅在缺少工具时安装。

### 4.3 检查读写权限

```bash
id
ls -l "$(readlink -f "$FOLLOWER_PORT")"
test -r "$FOLLOWER_PORT" && echo "serial readable"
test -w "$FOLLOWER_PORT" && echo "serial writable"
```

期望同时出现 `serial readable` 和 `serial writable`，且当前用户属于 `dialout` 组。本机 `oong7` 已加入该组。

如果缺少 `dialout`：

```bash
sudo usermod -aG dialout "$USER"
newgrp dialout
```

`newgrp` 会进入一个新的 Shell。接着执行 `id` 确认组权限，并重新设置 4.1 节的变量。也可以重新打开一个 Ubuntu 用户终端后检查。

不要用 `sudo python ...` 长期绕过权限问题，否则容易混入错误 Python 环境、创建 root 拥有的校准和日志文件。

### 4.4 确认串口没有被别的程序占用

```bash
fuser -v "$FOLLOWER_PORT"
pgrep -af '[c]artesian_control.py'
```

无进程占用时，`fuser` 通常不输出 PID，退出码为 1，这是正常结果。如果已有控制服务占用串口，应先使用已有页面或正常退出该服务。

同一串口一次只允许一个控制程序使用。不要同时运行校准、总线检查、释放力矩脚本和 Cartesian 服务。Windows 端的串口调试工具也应关闭。

### 4.5 权限仍异常时的定向 udev 规则

仅在设备已经存在、用户已加入 `dialout`，但实际设备节点权限仍不正确时考虑此分支。下面规则仅匹配当前这块控制板：

```bash
sudo tee /etc/udev/rules.d/99-so101-follower.rules >/dev/null <<'EOF'
SUBSYSTEM=="tty", ATTRS{idVendor}=="1a86", ATTRS{idProduct}=="55d3", ATTRS{serial}=="5AB0181922", GROUP="dialout", MODE="0660"
EOF
sudo udevadm control --reload-rules
sudo udevadm trigger --subsystem-match=tty
```

重新检查实际节点权限。如果 `udevadm` 提示服务未运行，先确认初始化系统：

```bash
ps -p 1 -o comm=
```

使用 systemd 时：

```bash
sudo systemctl start systemd-udevd
```

未使用 systemd、但提供 udev 服务脚本时：

```bash
sudo service udev start
```

只选择符合当前系统的启动方式，随后再重载规则。更换控制板时要同步修改序列号匹配条件。

## 5. 检查 Conda 环境和工程文件

### 5.1 已有环境的快速检查

本机环境已创建，日常无需重新安装依赖。

**执行位置：Ubuntu Bash，先执行 4.1 节变量设置。**

```bash
"$CARTESIAN_PY" --version
"$CARTESIAN_PY" -m pip check
```

检查主要包版本：

```bash
"$CARTESIAN_PY" - <<'PY'
from importlib.metadata import version
for name in ['lerobot', 'placo', 'viser', 'pin', 'cmeel-urdfdom', 'cmeel-tinyxml2']:
    print(f'{name}: {version(name)}')
PY
```

课程环境应包含：

```text
lerobot: 0.6.1
placo: 0.9.15
viser: 1.1.0
pin: 3.4.0
cmeel-urdfdom: 4.0.1
cmeel-tinyxml2: 10.0.0
```

本机另一个 `lerobot` 环境曾安装 0.6.2。不要因为名字相近就改用它，部分课程脚本会明确拒绝非 0.6.1 版本。

### 5.2 手动激活环境

启动脚本已经固定使用正确 Python，因此不激活 Conda 也能启动。需要调试、手动执行 Python 时可激活：

```bash
source ~/miniconda3/etc/profile.d/conda.sh
conda activate so101-cartesian
which python
cd ~/Lets-Build-Robots-S1-SO101-main
```

`which python` 应指向 `/home/oong7/miniconda3/envs/so101-cartesian/bin/python`。

### 5.3 确认模型和自定义脚本齐全

```bash
ls -l "$PROJECT_DIR/models/so101/so101_new_calib.urdf"
ls -l "$PROJECT_DIR/start-cartesian-gamepad.sh"
ls -l "$PROJECT_DIR/start-cartesian-real.sh"
ls -l "$PROJECT_DIR/start-cartesian-model.sh"
ls -l "$PROJECT_DIR/calibrate-cartesian-follower.sh"
ls -l "$PROJECT_DIR/Cartesian/browser_gamepad.py"
ls -l "$PROJECT_DIR/Cartesian/browser_gamepad.html"
```

浏览器手柄功能和这些启动脚本是本次加入的本地功能。只重新下载原始课程仓库，不会自动拥有这些修改；迁移时要备份当前工程版本，见第 11 节。

## 6. 从臂通信检查与重新校准

### 6.1 哪些情况需要校准

- 尚未生成本工程的从臂校准文件。
- 程序提示电机参数与校准文件不一致。
- 拆装了关节、电机或舵盘，或者在其他软件中改过偏移和行程。
- 更换了从臂，现有文件不属于这只机械臂。

正常的 USB 重连、电脑重启不要求重新校准。本机该文件已经生成并验证过匹配；不要每次启动都重做校准。

### 6.2 可选：只读检查 6 个电机

先结束所有占用串口的服务，确认机械臂已支撑好、电机电源正常。

```bash
cd "$PROJECT_DIR"
"$CARTESIAN_PY" Bringup/so101_bus_check.py \
  --role follower \
  --port "$FOLLOWER_PORT" \
  --samples 10 \
  --interval 0.2
```

期望看到 6 个电机在每次读取中都有响应，并以 `PASS` 结束。该命令不写电机寄存器，不验证真实运动。

如果提示 `Torque is already enabled`，表示该只读脚本检测到电机已有力矩，不是要求你打开力矩。先按第 9 节妥善停止、支撑并释放或断电，再检查。

### 6.3 备份已有校准

```bash
if [ -f "$PROJECT_DIR/calibration/follower/so101-follower.json" ]; then
  cp -- "$PROJECT_DIR/calibration/follower/so101-follower.json" \
    "$PROJECT_DIR/calibration/follower/so101-follower.json.$(date +%Y%m%d-%H%M%S).bak"
fi
```

旧缓存路径下的 `my_awesome_follower_arm.json` 不会因为运行本工程校准而被删除。不要为了消除报错，直接把不明来源的旧 JSON 覆盖进当前工程。

### 6.4 执行交互校准

先托住机械臂，校准程序会关闭力矩并写入校准参数。校准期间不要打开其他串口程序。

```bash
bash ~/Lets-Build-Robots-S1-SO101-main/calibrate-cartesian-follower.sh
```

终端交互顺序：

1. 出现 `Type CALIBRATE to continue:`，输入大写 `CALIBRATE`，回车。
2. 如果已有同 ID 的文件，会询问复用还是重做。**输入 `c` 再回车**才是重新校准；直接回车会使用已有文件。
3. 出现 `middle of its range of motion`，按校准示意把各关节摆到行程中位，然后回车。此时记录中位偏移。
4. 出现 `Recording positions` 后，依次缓慢移动 `shoulder_pan`、`shoulder_lift`、`elbow_flex`、`wrist_flex` 和 `gripper`，覆盖各自正常可动行程。夹爪要开合，不要只移动手臂。
5. 当前安装版本不采集 `wrist_roll` 的运动行程，它使用完整编码器范围；不要强行扭转手腕或线缆来补采。
6. 所有需要采集的关节都走过行程后，再按回车结束。
7. 等待出现 `Calibration saved to` 和保存路径。

中位姿态参考 [LeRobot SO101 校准视频](https://huggingface.co/docs/lerobot/main/en/so101#calibration-video)。视频用于理解摆放和移动方式；本机运行命令以本文固定的课程脚本为准。

不要在尚未移动各关节时就结束记录，也不要用蛮力推过机械限位。如果出现 `I/O error` 或 `STOP`，停止校准，先支撑机械臂并切断电机电源，再排查连接。

### 6.5 检查输出文件

```bash
ls -l "$PROJECT_DIR/calibration/follower/so101-follower.json"
"$CARTESIAN_PY" -m json.tool "$PROJECT_DIR/calibration/follower/so101-follower.json"
```

应含 6 个关节的校准数据。能解析 JSON 只代表文件格式有效；真实启动程序还会读取电机参数，与这份文件比较。

## 7. Windows：蓝牙连接手柄

### 7.1 在 Windows 中配对

1. 将飞智手柄切换到支持电脑使用的蓝牙模式，并按该型号说明进入配对状态。
2. 打开 Windows“设置 → 蓝牙和设备 → 添加设备 → 蓝牙”。
3. 选择手柄，等待连接完成。
4. 保持手柄连接 Windows；不要把内置蓝牙适配器转发给 Ubuntu。

本次设备在 Windows 中可能显示 Xbox 兼容名称，浏览器识别到：

```text
HID-compliant game controller (STANDARD GAMEPAD Vendor: 045e Product: 02e0)
```

这不要求手柄品牌名称一定显示为“飞智”。具体配对按键随型号变化，本文不猜测组合键。

### 7.2 验证 Windows 端有输入

按 `Win + R`，输入：

```text
joy.cpl
```

选择手柄，打开“属性”，检查摇杆和按钮响应。如果 Windows 自己都读不到输入，先处理手柄模式、蓝牙连接和电量，再检查 WSL 服务。

如果按键被改成鼠标／键盘行为，或被其他软件重新映射，先退出相关映射功能和测试工具，再重试标准游戏手柄模式。

### 7.3 为什么还要在网页上按一下 A

Windows 连接成功后，浏览器页面仍可能尚未获得该手柄的可见输入。保持控制页面在前台，按一下按钮或移动摇杆，可使页面识别到已连接设备；刷新页面后可能需要再次触发。[Gamepad API 说明](https://developer.mozilla.org/en-US/docs/Web/API/Gamepad_API/Using_the_Gamepad_API)

为避免误启动，识别时按 A 后松开即可，不要用 Start 测试。点击“连接手柄输入”前让所有按钮、扳机松开，摇杆回中。

## 8. 启动服务和浏览器控制

### 8.1 先确认没有重复服务

**执行位置：Ubuntu Bash。**

```bash
pgrep -af '[c]artesian_control.py'
ss -ltnp | grep -E ':(4602|4603|4604)\b'
fuser -v "$FOLLOWER_PORT"
```

如果已经有一套正确服务在运行，直接访问已有页面。日志中的历史 PID、本文旧记录或 `browser-gamepad-server.json` 中的启动快照，不能代替当前进程检查。

### 8.2 启动真实从臂

```bash
bash ~/Lets-Build-Robots-S1-SO101-main/start-cartesian-gamepad.sh --real
```

`--real` 要紧跟在脚本路径后面。本机脚本只有这个位置会被当作真实模式选择；省略它会启动模拟模式。

启动脚本会使用固定 Conda Python，指定稳定串口、从臂 ID 和校准目录。真实模式参数包括：

| 参数                             | 当前值    |
| -------------------------------- | --------- |
| 真实页面                         | 4603      |
| 手柄统一入口                     | 4604      |
| 控制循环默认频率                 | 30 Hz     |
| 5 个手臂关节的速度上限           | 0.5 rad/s |
| 手臂命令相对测量位置的领先角上限 | 5°        |

夹爪采用独立速度与领先量限制；不要把手臂的 0.5 rad/s 当作夹爪速度。

期望启动日志出现：

```text
arm connected
open http://127.0.0.1:4603
Windows gamepad page: http://127.0.0.1:4604
loop start
```

服务连接串口并读取位置，网页默认不启用控制来源，也不自动执行 `Hold and follow`。但启动程序本身不会强制清除先前其他程序留下的电机力矩，因此不能仅凭“刚启动”推断物理上一定无力矩。

### 8.3 从 Windows 直接启动的等价命令

如果不想先进入 Ubuntu，可以在 Windows PowerShell 直接执行：

```powershell
wsl -d Ubuntu-24.04 -u oong7 -- bash /home/oong7/Lets-Build-Robots-S1-SO101-main/start-cartesian-gamepad.sh --real
```

这个终端会持续显示服务日志。不要再次到 Ubuntu 启动第二份。

### 8.4 打开统一入口并连接手柄

**执行位置：Windows Chrome。**

`http://127.0.0.1:4604`

操作顺序：

1. 看上方“当前目标”，真实模式应显示真实从臂。
2. 如果手柄尚未出现，保持页面在前台，按一下 A 后松开。
3. 如检测到多个手柄，在上方下拉框选择本次使用的设备。
4. 让摇杆回中，松开所有按钮和扳机。
5. 点击“连接手柄输入”。
6. 等待手柄名称和递增的“已接收 N 帧”。
7. 下方面板的 `Gamepad` 状态应显示手柄名称，`Enable Gamepad` 由灰色变为可点击。控制程序每秒检查一次新设备，因此可能需要等一个检查周期。

浏览器要求 `standard` 映射、至少 4 个摇杆轴和 17 个按钮。非标准模式会被页面拒绝；不要在不确定编号时直接让真实机械臂试动作。

### 8.5 开始控制真实从臂

1. 确认下方 `Follower = Real arm`。
2. 确认 `Mode = Gamepad`。浏览器手柄启动模式会默认选好；如果使用了其他启动方式，需自行选择。
3. 点击 `Enable Gamepad`。
4. 确认现场准备完成，摇杆仍在中位。
5. 点击 `Hold and follow`，或按 Start。
6. 轻推一个摇杆方向，再松开，检查运动方向和停止行为。
7. 再分别验证底座、俯仰、滚转和夹爪，首次短暂操作。

`Enable Gamepad` 启用输入来源；`Hold and follow` 才开启力矩并开始跟随。它会先以当前测量位置设置保持目标，减少启动时因旧目标造成的跳动。

### 8.6 手柄映射

| 手柄操作     | 动作                     |
| ------------ | ------------------------ |
| 左摇杆前／后 | 末端升高／降低           |
| 左摇杆右／左 | 末端伸出／收回           |
| 右摇杆前／后 | 末端俯仰                 |
| 右摇杆左／右 | 手腕滚转                 |
| LT／RT       | 底座向左／向右转         |
| 按住 B       | 夹爪打开，松开停止       |
| 按住 X       | 夹爪闭合，松开停止       |
| Start        | 相当于 `Hold and follow` |
| Back／Select | 相当于 `Stop`，保持力矩  |

这是当前工程的 Xbox 标准布局名称。其他外观按键按标准布局映射，不以按键上印刷文字自动判断。

正常标准映射会自动生成，不需要十步配对向导。映射文件独立存放在：

```text
/home/oong7/Lets-Build-Robots-S1-SO101-main/Cartesian/gamepad_map.browser.json
```

如果误用了 `Pair again`，可先停止控制，再按第 10.11 节备份映射并重新建立默认布局。

### 8.7 先练习模拟机械臂

结束真实服务后，在 Ubuntu 启动：

```bash
bash ~/Lets-Build-Robots-S1-SO101-main/start-cartesian-gamepad.sh
```

仍访问 4604。此时页面内部嵌入的是 4602，Follower 为 `Simulated arm`，不打开真实串口。

真实服务和模拟服务默认都使用 4604，因此不要同时启动两份。练习完 Ctrl+C 退出，再用 `--real` 启动真实服务，并刷新网页。

### 8.8 改用鼠标拖拽

使用同一真实页面时，先停止跟随，再禁用当前 Gamepad 来源，切换 Mode 到 `Drag to move`，启用该来源后再手动开始跟随。

也可以结束当前服务后，仅启动原有鼠标控制页面：

```bash
bash ~/Lets-Build-Robots-S1-SO101-main/start-cartesian-real.sh
```

访问 `http://127.0.0.1:4603`。这个命令没有启动浏览器手柄桥接，不能期待 4604 可用。

## 9. 停止、退出、断电和恢复

### 9.1 各种停止方式的区别

| 操作                         | 停止跟随                           | 关闭力矩                     | 结束进程       |
| ---------------------------- | ---------------------------------- | ---------------------------- | -------------- |
| Back／Select                 | 是                                 | 否                           | 否             |
| 下方 `Stop`                  | 是                                 | 否                           | 否             |
| 上方“停止手柄输入”           | 连接失效后停止，含控制循环处理时间 | 否                           | 否             |
| 手柄断开／页面失焦／输入超时 | 连接失效后停止，含控制循环处理时间 | 否                           | 否             |
| 禁用当前控制来源             | 是                                 | 否                           | 否             |
| `Release torque`             | 是                                 | 尝试关闭；通信故障时可能失败 | 否             |
| 默认 Ctrl+C                  | 退出控制循环                       | 默认保留                     | 是             |
| 切断电机电源                 | 电机失去供电                       | 是，且失去支撑               | 服务不一定退出 |

不要把 `Stop`、关闭浏览器或 Ctrl+C 当作断电。软件停止也不能代替现场可用的物理电源切断。

### 9.2 日常结束顺序

1. 松开摇杆、扳机和夹爪按键。
2. 按 Back 或点击 `Stop`，确认停止跟随。
3. 托住机械臂或放置到可支撑的位置。
4. 点击 `Release torque`，确认机械臂状态。
5. 在运行服务的终端按 Ctrl+C。
6. 需要改线或收纳时切断从臂电机电源，再处理线缆。
7. 如需交回 Windows 使用控制板，再执行 `usbipd detach`。

### 9.3 页面失焦、手柄断开后的恢复

当前实现会在这些情况下停止手柄输入：

- 切到另一个应用，浏览器文档失去焦点。
- 切换标签页、隐藏页面。
- 手柄关闭或蓝牙断开。
- 浏览器采样出现长于约 350 ms 的调度间隔。
- 服务超过 0.4 秒没有收到有效输入。

超时阈值不等于严格保证的物理停止时间，还要加上线程调度和控制循环处理时间。页面菜单、系统弹窗或电脑休眠也可能触发保护。

恢复顺序：

1. 回到控制页面并保持前台。
2. 如刚重连蓝牙或刷新了网页，先按 A 后松开。
3. 所有控件回到静止状态。
4. 点击“连接手柄输入”。
5. 必要时重新点击 `Enable Gamepad`。
6. 准备好后重新点击 `Hold and follow` 或按 Start。

恢复数据连接不会自动恢复运动。停止和释放按钮在输入来源失效后仍应可操作；Hold 在没有有效来源时保持禁用。

### 9.4 正常结束后台或其他终端的服务

优先回到原来运行服务的终端，使用 Ctrl+C。若找不到原终端：

```bash
pgrep -af '[c]artesian_control.py'
ss -ltnp | grep -E ':(4602|4603|4604)\b'
```

先在页面停止、托住并释放力矩，再选择刚查到的正确 PID：

```bash
read -r -p "输入刚核对过的 Cartesian 进程 PID: " CARTESIAN_PID
ps -p "$CARTESIAN_PID" -o pid,user,args
```

确认该进程确实属于当前工程的 `cartesian_control.py` 后，执行：

```bash
kill -INT "$CARTESIAN_PID"
```

再用 `pgrep` 和 `ss` 检查是否退出。不要照抄历史 PID，也不要用 `pkill python` 或 `killall python`，以免影响其他程序。

进程或通信已经卡死时，先托住并切断电机电源，再处理程序；不要假设强制结束进程会关闭力矩。

### 9.5 独立释放力矩脚本

用于页面不可用、原程序退出后机械臂仍保持力矩的情况。先托住机械臂，并确认占用串口的控制程序已经结束。

```bash
fuser -v "$FOLLOWER_PORT"
cd "$PROJECT_DIR"
"$CARTESIAN_PY" Bringup/so101_release_torque.py \
  --port "$FOLLOWER_PORT" \
  --verify-samples 3 \
  --verify-interval 1
```

这个命令会真实写入 6 个电机的关闭力矩寄存器，再多次读回检查，不能与控制服务同时运行。

成功输出包含：

```text
REGISTER CHECK: all six motors report Torque_Enable=0 across the verification samples.
```

这只证明寄存器读回，不是物理力矩测量。如果仍僵硬、报 `UNKNOWN` / `NO REPLY` / `FAIL`，或者无法打开串口，托住机械臂并切断电机电源。

### 9.6 将控制板交回 Windows

**执行位置：Windows PowerShell；确认控制程序已停止。**

```powershell
usbipd list
$armBusId = '2-3'
usbipd detach --busid $armBusId
```

如果曾使用强制绑定，Windows 仍不能使用设备时，在管理员 PowerShell 解除绑定：

```powershell
usbipd unbind --busid $armBusId
```

下次要给 WSL 使用时，重新 `bind` 和 `attach`。USB/IP detach 不会关闭从臂电机电源。

## 10. 常见故障排查

### 10.1 `usbipd` 找不到、服务未运行或要求管理员权限

Windows PowerShell 检查：

```powershell
Get-Command usbipd
Get-Service usbipd
```

安装后命令仍找不到时，重新打开终端。服务确实停止时，在管理员 PowerShell 启动：

```powershell
Start-Service usbipd
```

`Access denied; this operation requires administrator privileges` 指 Windows 终端未提升权限。右键以管理员身份打开 PowerShell 后，再执行 `bind`；在 WSL 中加 `sudo` 无效。

### 10.2 `Device busy (exported)` / Windows 正在使用设备

先重新执行 `usbipd list`，确认报错 BUSID 现在对应从臂控制板，不能根据旧编号直接强制绑定。本机 `2-2` 曾先后是鼠标接收器和手柄，已经出现过编号变化。

关闭占用该控制板的 Windows 串口软件，再尝试 attach。仍失败且确认目标无误时，在管理员 PowerShell：

```powershell
$armBusId = '2-3'
usbipd bind --busid $armBusId --force
usbipd attach --wsl --busid $armBusId
usbipd list
```

`--force` 属于 `bind`，不是 `attach`。强制绑定会使 Windows 无法使用这块设备，必要时通过 `unbind` 恢复。

本机也报告过 `Unknown USB filter 'hrdevmon'`；这是可能存在第三方 USB 过滤驱动的提示，不应据此随意卸载驱动或关闭整机保护。先按上面的目标核对和单设备绑定流程处理。

### 10.3 Windows 显示 Attached，但 Ubuntu 没有串口

Ubuntu 检查：

```bash
ls -l /dev/serial/by-id/
ls -l /dev/ttyACM*
dmesg | tail -n 80
```

如果 `dmesg` 拒绝访问，用 `sudo dmesg | tail -n 80`。检查 USB/IP 断开、驱动加载或设备复位信息。

若根本没有设备，确认接收转发后设备仍在、USB 数据线支持数据传输、控制板 USB 供电正常。在控制已停止、机械臂已支撑的前提下，Windows 重新 detach / attach：

```powershell
usbipd detach --busid $armBusId
usbipd attach --wsl --busid $armBusId
```

如果只出现 `/dev/ttyACM0`，但没有 `by-id`，继续检查 udev：

```bash
udevadm info --query=property --name=/dev/ttyACM0
```

确认 `ID_SERIAL` 和板子身份后恢复 udev / 稳定链接。不要在多控制板场景盲目把所有路径替换成 ACM0。

### 10.4 `Follower serial port missing or inaccessible` / `Permission denied`

依次检查：设备是否存在 → 稳定路径是否属于当前控制板 → 用户是否属于 dialout → 节点是否可读写。使用第 4 节命令。

更换控制板导致序列号改变时，先保存并确认新路径。可用命令找出脚本中固定的旧值：

```bash
grep -n 'FOLLOWER_PORT=' "$PROJECT_DIR/calibrate-cartesian-follower.sh" "$PROJECT_DIR/start-cartesian-real.sh"
```

然后编辑这两个文件中的 `FOLLOWER_PORT`：

```bash
nano "$PROJECT_DIR/calibrate-cartesian-follower.sh"
nano "$PROJECT_DIR/start-cartesian-real.sh"
```

改为实际 `/dev/serial/by-id/...` 路径，保存后检查语法：

```bash
bash -n "$PROJECT_DIR/calibrate-cartesian-follower.sh"
bash -n "$PROJECT_DIR/start-cartesian-real.sh"
```

`nano` 未安装时可使用自己熟悉的编辑器。新板连接的必须仍是同一只从臂，否则不能沿用原校准。

### 10.5 `I/O error` / 电机读写失败 / 六个电机均无法释放

先停止动作并支撑机械臂，必要时直接切断电机电源。不要在重复失败时连续尝试 Hold。

分两层排查：

| 层次                 | 检查内容                                                                             |
| -------------------- | ------------------------------------------------------------------------------------ |
| Windows → WSL USB/IP | `usbipd list` 是否仍 Attached，Ubuntu `dmesg` 是否有断开／复位，USB 线和接口是否可靠 |
| 控制板 → 电机总线    | 电机电源是否正常，总线线缆是否松动，电机 ID 和波特率是否与原配置一致                 |

故障恢复后，先确认串口无占用、只读检查能通过，再启动控制。USB/IP 连接中断时，软件的力矩释放命令也可能无法送达。

### 10.6 校准文件不存在或与电机不一致

报错可能包含：

```text
Follower calibration not found
no calibration for id ...
motors ... do not carry the follower calibration
```

先确认从臂串口没有与主臂混淆，再按第 6 节重新校准并保存到当前工程路径。不要把旧缓存文件重命名后强行绕过匹配检查。

### 10.7 `Use lerobot==0.6.1` / `ModuleNotFoundError`

检查是否误用了其他 Conda 环境或系统 Python：

```bash
which python
"$CARTESIAN_PY" -c "from importlib.metadata import version; print(version('lerobot'))"
"$CARTESIAN_PY" -m pip check
```

优先使用本文启动脚本。缺少模型或依赖时按第 12 节恢复；不要把另一个环境全局降级，也不要直接升级所有包。

### 10.8 4604 打不开或连接被拒绝

先看启动终端是否已经报错退出，再在 Ubuntu 检查：

```bash
ss -ltnp | grep -E ':(4602|4603|4604)\b'
"$CARTESIAN_PY" - <<'PY'
import urllib.request
with urllib.request.urlopen('http://127.0.0.1:4604/status', timeout=3) as response:
    print(response.read().decode())
PY
```

若 Ubuntu 能访问，在 Windows PowerShell 检查：

```powershell
curl.exe --noproxy "*" http://127.0.0.1:4604/status
```

- `Connection refused`：优先检查服务是否在运行、端口是否一致。
- 普通页面能打开但使用代理后失败：让 localhost / 127.0.0.1 绕过代理，或用上面的无代理命令对比。
- 如果只启动了 `start-cartesian-real.sh`，4604 本来就不会启动；应使用 `start-cartesian-gamepad.sh --real`。
- 不要为了排查本机页面而把服务改成监听 `0.0.0.0` 或关闭 Windows 防火墙。当前方案使用本机回环地址。

### 10.9 `Address already in use` / 网页端口被占用

```bash
ss -ltnp | grep -E ':(4602|4603|4604)\b'
pgrep -af '[c]artesian_control.py'
```

优先使用或正常退出已有服务，按第 9.4 节处理。

确实需要换端口时，可在没有串口竞争的前提下使用：

```bash
bash ~/Lets-Build-Robots-S1-SO101-main/start-cartesian-gamepad.sh --real \
  --web-port 4613 \
  --gamepad-web-port 4614
```

访问 `http://127.0.0.1:4614`。嵌入页面会跟随指定的控制端口。换网页端口不能解决同一从臂串口被其他进程占用的问题。

### 10.10 Windows 蓝牙已连接，但页面显示未检测到手柄

1. 用 Windows 的 `joy.cpl` 确认输入正常。
2. 使用 Windows Chrome 打开 4604，不要在 WSL 内另开 Linux 浏览器读取 Windows 手柄。
3. 把页面放前台，按 A 后松开，再点击“连接手柄输入”。
4. 检查手柄是否进入鼠标／键盘模式；必要时按该型号说明切换为电脑标准游戏手柄模式。
5. 暂停其他手柄映射或测试软件，重新检查。
6. 若有多个同地址页面，停止或关闭旧页面的手柄输入，只保留一个控制入口。

提示“非标准映射”时，当前页面不会猜测轴编号。切换到浏览器能报告 `mapping = standard` 的模式后再连接。

### 10.11 已连接输入，但 `Enable Gamepad` 灰色或映射异常

先等待一个设备检查周期，确认接收帧数持续递增。如果显示“采样暂停”“失去焦点”或“输入连接已停止”，按第 9.3 节恢复。

如果手动执行过 `Pair again` 或映射文件被改坏，可在停止跟随、退出控制服务后备份映射：

```bash
if [ -f "$PROJECT_DIR/Cartesian/gamepad_map.browser.json" ]; then
  mv -- "$PROJECT_DIR/Cartesian/gamepad_map.browser.json" \
    "$PROJECT_DIR/Cartesian/gamepad_map.browser.json.$(date +%Y%m%d-%H%M%S).bak"
fi
```

再重新启动服务、刷新页面并连接手柄，标准布局会自动重建。无需修改旧的 `Cartesian/gamepad_map.json`。

### 10.12 手柄输入正常，但真实机械臂不动

依次核对：

1. 启动命令是否带 `--real`。
2. `Follower` 是否为 `Real arm`。
3. `Mode` 是否为 `Gamepad`，且已经点击 `Enable Gamepad`。
4. 是否点击过 `Hold and follow` 或按过 Start。
5. 是否因页面失焦、数据超时而重新停止，需要手动恢复。
6. 目标是否超出可达范围、碰到软件边界或实际机械约束。
7. 电机是否报错、供电是否正常、串口是否仍连通。

红色不可达目标提示并不等同于串口故障。不要通过持续大幅推杆来解决不可达目标，先回到小幅可达动作检查。

### 10.13 不明电机故障、碰撞或明显异常运动

先停止并支撑，必要时切断电机电源，再保存日志排查。本工程是运动学控制演示；模型加载时的自碰撞提示或测试通过，不代表具备经过验证的防碰撞能力。

## 11. 日志、检查与备份

### 11.1 查看最新控制日志

**执行位置：Ubuntu Bash。**

```bash
cd "$PROJECT_DIR"
readlink -f Cartesian/logs/latest
tail -n 100 Cartesian/logs/latest/run.log
tail -n 3 Cartesian/logs/latest/ticks.jsonl
```

需要持续观察时：

```bash
tail -f Cartesian/logs/latest/run.log
```

在这个日志查看终端按 Ctrl+C，只结束 `tail`，不会结束另一个终端中的控制服务。观察日志会使浏览器失焦时，手柄输入可能按设计停止；恢复方法见第 9.3 节。

### 11.2 查看手柄输入状态

Ubuntu：

```bash
"$CARTESIAN_PY" - <<'PY'
import json
import urllib.request
with urllib.request.urlopen('http://127.0.0.1:4604/status', timeout=3) as response:
    print(json.dumps(json.load(response), ensure_ascii=False, indent=2))
PY
```

主要字段：

| 字段               | 含义                         |
| ------------------ | ---------------------------- |
| `connected`        | 最近手柄输入是否仍有效       |
| `name`             | 浏览器报告的设备名称         |
| `frames`           | 当前服务已接收的有效输入帧数 |
| `age_ms`           | 最近有效帧距现在的时间       |
| `axes` / `buttons` | 当前输入；失效时返回中立状态 |

从浏览器切换到终端查看状态可能触发失焦停止，因此看到 `connected: false` 时，要结合当时窗口焦点判断，不要直接认定蓝牙断了。

### 11.3 读取最新一帧控制状态

下列命令只解析日志，不连接串口：

```bash
"$CARTESIAN_PY" - <<'PY'
import json
from pathlib import Path
path = Path.home() / 'Lets-Build-Robots-S1-SO101-main/Cartesian/logs/latest/ticks.jsonl'
with path.open('rb') as stream:
    stream.seek(0, 2)
    length = stream.tell()
    stream.seek(max(0, length - 65536))
    rows = stream.read().splitlines()
for row in reversed(rows):
    try:
        state = json.loads(row)
        break
    except json.JSONDecodeError:
        continue
else:
    raise SystemExit('No complete tick yet')
for key in ['follower', 'mode', 'following', 'torque', 'q_meas_deg']:
    print(key, state.get(key))
PY
```

这些是软件最近一帧状态。尤其是 `torque`，不能当作独立实时的物理力矩测量或代替电机寄存器读回。

### 11.4 不连接硬件的回归测试

```bash
cd "$PROJECT_DIR"
"$CARTESIAN_PY" Cartesian/test_cartesian.py --model-dir models/so101
"$CARTESIAN_PY" Cartesian/test_browser_gamepad.py
```

本次最终版本原有 39 项和新增 14 项通过。测试会启动本机临时服务、使用模拟电机并写测试日志；不执行真实机械臂动作。正常使用不需要每次运行整套测试。

### 11.5 备份当前工程修改和校准

先停止控制服务，避免边运行边备份校准和映射文件：

```bash
cd "$PROJECT_DIR"
tar -czf "$HOME/so101-cartesian-config-$(date +%Y%m%d-%H%M%S).tar.gz" \
  --exclude='Cartesian/logs' \
  --exclude='*/__pycache__' \
  Cartesian Bringup calibration \
  start-cartesian-gamepad.sh start-cartesian-real.sh start-cartesian-model.sh \
  calibrate-cartesian-follower.sh \
  environment-cartesian.yml requirements-cartesian.txt requirements-cartesian.lock.txt \
  SO101_Windows到WSL_蓝牙手柄实机控制完整操作手册.md
```

此包包含本地修改、校准和映射，不包含整个 Conda 环境和 `models/so101`。模型可按第 12 节重新下载；环境可按导出文件恢复。

校准属于这只实际机械臂，不应直接应用到另一只外观相同的机械臂。

## 12. 环境损坏或迁移后的恢复

### 12.1 恢复丢失的模型文件

**只在模型缺失时执行，Ubuntu Bash：**

```bash
cd "$PROJECT_DIR"
"$CARTESIAN_PY" Cartesian/fetch_model.py --model-dir models/so101
```

脚本从固定提交下载 URDF、网格和许可文件；已存在且非空的文件会保留。需要能访问 GitHub 的网络。

### 12.2 用 Conda 导出重建环境

仅当 `so101-cartesian` 环境尚不存在时执行，不要在正常环境上重复创建或先删除现有环境：

```bash
source ~/miniconda3/etc/profile.d/conda.sh
cd "$PROJECT_DIR"
conda env create -f environment-cartesian.yml
conda activate so101-cartesian
python -m pip check
```

本机历史安装记录中环境约 6 GiB，此外还有模型、下载缓存和日志。导出面向 Linux；不要直接拿到 Windows 原生 Python 中安装 placo 等 Linux 依赖。

### 12.3 导出环境无法重建时的核心版本安装方式

同样只在目标环境尚不存在时创建；先保留现有可用环境和备份，再按需恢复：

```bash
source ~/miniconda3/etc/profile.d/conda.sh
conda create -n so101-cartesian python=3.12 -y
conda activate so101-cartesian
python -m pip install \
  'lerobot[kinematics,feetech]==0.6.1' \
  'viser[urdf]==1.1.0' \
  'placo==0.9.15' \
  'pin==3.4.0' \
  'cmeel-urdfdom==4.0.1' \
  'cmeel-tinyxml2==10.0.0'
python -m pip check
```

核心依赖安装不能保证所有传递依赖与此前完全相同。需要更接近原环境时，结合 `requirements-cartesian.lock.txt` 和完整导出进行核对，最后运行模型及手柄测试。

### 12.4 代理造成安装或下载缓慢

先查看是否配置了代理，不要把带账号、口令的代理地址贴到公开日志：

```bash
env | grep -iE '^(http_proxy|https_proxy|all_proxy|no_proxy)='
```

只在已确认直连官方 PyPI 可用、而代理链路异常时，可对一次安装命令临时绕过代理，不修改全局配置：

```bash
env -u HTTP_PROXY -u HTTPS_PROXY -u ALL_PROXY \
    -u http_proxy -u https_proxy -u all_proxy \
    NO_PROXY='*' no_proxy='*' \
    "$CARTESIAN_PY" -m pip install -r "$PROJECT_DIR/requirements-cartesian.txt"
```

GitHub 模型下载是否需要代理要单独判断，不要因为 PyPI 直连更快就全局关闭代理。

### 12.5 重启 WSL 的影响

只有完成停止、支撑和断电后，才考虑用以下命令恢复 WSL 故障：

```powershell
wsl --shutdown
```

这会关闭所有 WSL 发行版和其中的服务，USB/IP 需要重新 attach。不要在机械臂运行中用它当作常规停止方法。之后从第 3 节重新建立串口转发，再启动工程。

## 13. 文件与验证记录

### 13.1 主要文件

| 路径，相对工程根目录                       | 用途                                                   |
| ------------------------------------------ | ------------------------------------------------------ |
| `start-cartesian-gamepad.sh`               | 统一手柄启动入口；无参数为模拟，首参数 `--real` 为真实 |
| `start-cartesian-real.sh`                  | 真实从臂参数、稳定串口和较低速度配置                   |
| `start-cartesian-model.sh`                 | 无串口的模型启动入口                                   |
| `calibrate-cartesian-follower.sh`          | 现场交互校准入口                                       |
| `Bringup/so101_bus_check.py`               | 只读检查 6 个电机                                      |
| `Bringup/so101_release_torque.py`          | 独立释放力矩与寄存器读回检查                           |
| `Cartesian/browser_gamepad.py`             | 本机输入服务与超时处理                                 |
| `Cartesian/browser_gamepad.html`           | Windows 手柄入口与嵌入式控制页面                       |
| `Cartesian/gamepad.py`                     | 手柄后端，包括 browser / Linux jsdev / pygame          |
| `Cartesian/gamepad_map.browser.json`       | 浏览器手柄布局；与原 Linux 映射分开                    |
| `calibration/follower/so101-follower.json` | 当前从臂校准                                           |
| `Cartesian/logs/latest/run.log`            | 最近服务的事件与故障日志                               |
| `Cartesian/logs/latest/ticks.jsonl`        | 控制循环逐帧记录                                       |

### 13.2 Windows 访问 Ubuntu 工程

在 Windows 文件资源管理器地址栏输入：

```text
\\wsl.localhost\Ubuntu-24.04\home\oong7\Lets-Build-Robots-S1-SO101-main
```

工程中的 Markdown 可直接阅读。不要在服务运行中修改启动参数、校准或手柄映射文件。

### 13.3 已验证范围与边界

截至本次配置记录，已验证：

- Ubuntu-24.04 中的课程 Conda 环境和模型可以加载。
- 原有 39 项测试与新增 14 项测试通过。
- Windows Chrome 能读到实际蓝牙手柄，并向 WSL 传输数据。
- 一次实机待机采样约 31 帧/秒，31 个采样点均连接正常，最大数据年龄 48 ms；这是该次测量，不是长期实时性保证。
- 当前从臂校准曾与 6 个电机参数匹配。
- 真实服务能够建立串口连接、读取位置并保持待机。
- 软件测试验证了输入失效、禁用来源后的停止跟随，以及停止／释放按钮状态。

这些记录不代表此后设备永远在线、当前力矩一定关闭，或真实运动、抓取和碰撞保护已经完成验证。每次现场启动仍应核对真实页面、设备状态与小幅动作。

### 13.4 参考资料

- [微软：连接 USB 设备到 WSL](https://learn.microsoft.com/en-us/windows/wsl/connect-usb)
- [MDN：使用 Gamepad API](https://developer.mozilla.org/en-US/docs/Web/API/Gamepad_API/Using_the_Gamepad_API)
- [LeRobot：SO101 校准示意](https://huggingface.co/docs/lerobot/main/en/so101#calibration-video)

串口路径、脚本参数、按键映射和超时阈值以本机源码核对结果为准；上游 main 文档更新不自动改变当前工程的固定依赖和实现。
