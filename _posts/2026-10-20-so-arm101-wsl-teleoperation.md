---
layout: post
title: SO-ARM101：WSL 检查与主从臂遥操作指南
date: 2026-10-20 00:00:00 +0800
description: 记录 SO-ARM101 在 WSL2 下的环境检查、USB/IP 转接、串口识别、电机校准与主从臂遥操作流程。
tags: [LeRobot, SO-ARM101, WSL, 机器人]
categories: [机器人开发]
published: true
toc:
  sidebar: left
---

检查日期：2026-10-20（北京时间）。硬件配置：用户确认已有 Leader 主臂与 Follower 从臂。本文先完成两臂连接同一台电脑的遥操作；跨网络遥操作需要另行设计通信链路。

## 1. 本机实测结果

| 项目             | 实测结果                                                                                       | 判断                                       |
| ---------------- | ---------------------------------------------------------------------------------------------- | ------------------------------------------ |
| WSL              | Ubuntu-24.04，运行于 WSL 2                                                                     | 正常                                       |
| Ubuntu           | 24.04.4 LTS，x86_64，内核 6.18.33.2-microsoft-standard-WSL2                                    | 正常                                       |
| 用户             | oong7，默认 shell 为 zsh                                                                       | 已确认                                     |
| 内存             | WSL 配置上限 16 GB，Linux 报告约 15 GiB                                                        | 足够开始遥操作开发                         |
| 磁盘             | Linux 文件系统报告剩余约 944 GB；宿主 C 盘实际剩余约 330 GB                                    | 虚拟磁盘显示容量不能视为宿主可用空间       |
| Conda            | /home/oong7/miniconda3，Conda 26.9.3                                                           | 已安装                                     |
| 开发环境         | /home/oong7/miniconda3/envs/lerobot，Python 3.12.15                                            | 已存在，无需新建                           |
| LeRobot          | 0.6.2，以 editable 方式安装，源码 /home/oong7/lerobot                                          | 已安装                                     |
| 源码版本         | 146d8474，2026-10-16；检查结束时 git status 无修改                                             | 本次未更新源码                             |
| PyTorch          | 2.11.0+cu130，torchvision 0.26.0+cu130                                                         | 导入成功                                   |
| GPU              | RTX 2080 Super，8 GB，驱动 610.74                                                              | WSL 内可见                                 |
| GPU 运算         | CUDA 创建张量、平方求和得到 1240.0，并成功同步                                                 | 基础计算实测通过；不代表训练已验证         |
| 电机基础依赖     | rustypot 1.13.1、pyserial 3.5                                                                  | 导入成功                                   |
| SO101 类         | 主臂、从臂导入及对象创建成功，未调用 connect                                                   | 未接触真实电机                             |
| CLI              | lerobot-teleoperate、lerobot-calibrate 的 --help 返回 0                                        | 命令入口正常                               |
| 包依赖一致性     | pip check：No broken requirements found                                                        | 只检查已安装依赖，不保证可选功能完整       |
| 缺少的可选依赖   | evdev、pynput、deepdiff、rerun-sdk、datasets、av、torchcodec；Conda 环境没有 ffmpeg 可执行文件 | 完整硬件工作流、显示、录像需要补齐相关依赖 |
| 串口权限         | oong7 尚不属于 dialout，dialout 组无成员                                                       | 连接后应配置权限                           |
| USB/IP           | usbipd-win 5.3.0，服务 Running、Automatic                                                      | 已安装                                     |
| Windows USB 设备 | 当前只有鼠标接收器、HP 摄像头、蓝牙，未看到机械臂控制板                                        | 需要插入两块控制板后重新检查               |
| Linux 设备       | 没有 /dev/ttyACM*、/dev/ttyUSB*、/dev/video\*                                                  | 尚未完成机械臂/摄像头转接                  |
| 其他串口         | Windows COM6/7/12/13 为蓝牙串口；Linux 枚举到 ttyS0–7                                          | 不能据此认定找到了机械臂                   |
| 校准             | 默认缓存位置未发现校准 JSON 文件                                                               | 本机尚未验证校准完成                       |
| 网络             | Ubuntu 对 PyPI、conda-forge 和 GitHub 的 HTTPS HEAD 请求均返回 200                             | 基础访问正常；没有验证大文件完整下载       |

本次完成环境检查，没有安装/升级包、修改用户权限、绑定 USB 设备、改写电机参数或启动机械臂运动。检查时执行了导入、对象创建及 CLI 帮助；这些操作可能生成 Python 缓存和空校准目录。

## 2. 进入现有 Conda 环境

在 Windows PowerShell 中：

```powershell
wsl -d Ubuntu-24.04
```

以下命令均在 Ubuntu 终端执行，除非明确标为 PowerShell：

```bash
source /home/oong7/miniconda3/etc/profile.d/conda.sh
conda activate lerobot
cd /home/oong7/lerobot
which python
python --version
```

解释器应是 `/home/oong7/miniconda3/envs/lerobot/bin/python`。Conda 已初始化到 zsh；非交互 bash 中找不到 conda 不代表没有安装，显式 source 即可。

继续使用 Linux 中的 `/home/oong7/lerobot` 开发。若使用 VS Code，连接 WSL: Ubuntu-24.04，打开该目录，再选择上述解释器。

## 3. 按用途补齐依赖

仅做主从控制，先补硬件工作流依赖：

```bash
python -m pip install -e '.[hardware,feetech]'
conda install -c conda-forge evdev
python -m pip check
```

本机基础控制类和 CLI 已经能加载；缺少的录像/显示包不能直接等同于“纯主从控制必然失败”。补装 hardware 是使声明的硬件依赖完整，evdev 是官方针对 WSL 的安装建议。

以后需要摄像头显示、录制演示数据时，再补：

```bash
python -m pip install -e '.[core_scripts,feetech]'
conda install -c conda-forge ffmpeg
python -m pip check
```

如遇 TorchCodec/FFmpeg 加载错误，应按实际安装版本检查兼容关系。当前 GPU 基础运算已通过，没有理由仅因它使用 CUDA 13 就重装 PyTorch。纯主从遥操作不依赖训练模型，也不需要先配置 Hugging Face 登录。

官方参考：[LeRobot 安装说明](https://huggingface.co/docs/lerobot/installation)。

## 4. 将两个控制板转接给 WSL

先插入主臂与从臂控制板的 USB 数据线，并按各自套件规格供电。区分两块板，确认哪块是 Leader、哪块是 Follower。

管理员 PowerShell 中查看设备：

```powershell
usbipd list
```

找到新增的两个控制板 BUSID。下面的占位内容必须替换为实际 BUSID；不要把鼠标、蓝牙设备当作机械臂板绑定：

```powershell
usbipd bind --busid <主臂BUSID>
usbipd bind --busid <从臂BUSID>
```

保持 Ubuntu 运行，再执行 attach；已 bind 的设备 attach 通常不需要管理员权限：

```powershell
usbipd attach --wsl --busid <主臂BUSID>
usbipd attach --wsl --busid <从臂BUSID>
usbipd list
```

应显示 Attached。设备转给 WSL 后，Windows 不能同时使用它。拔插设备或重启 WSL 后，应重新检查并按需 attach。

本机 usbipd 报告存在 `hrdevmon` USB filter，可能影响兼容性。这只是检查时的提示；先用普通 bind，若确实失败，再依据错误定位过滤驱动冲突。

回到 Ubuntu：

```bash
ls -l /dev/ttyACM* /dev/ttyUSB*
ls -l /dev/serial/by-id/
```

根据控制板 USB 芯片，设备可能是 ttyACM，也可能是 ttyUSB。Linux 尚未安装 lsusb 命令，需要时执行 `sudo apt-get install usbutils`。

配置用户权限：

```bash
sudo usermod -aG dialout "$USER"
```

重新登录 Ubuntu 后执行 `id`，确认含 dialout，并重新激活 Conda。已有会话若未刷新组身份，可以先执行 `newgrp dialout` 再激活环境。以 `ls -l` 显示的设备属组和权限为准；若仍无访问权，检查 udev 规则。

官方参考：[连接 USB 设备到 WSL](https://learn.microsoft.com/en-us/windows/wsl/connect-usb)。

## 5. 固定主从端口

不要假设 ttyACM0 一定是从臂。分别接入控制板，结合 Windows BUSID、Linux 设备路径确认对应关系。优先使用可唯一识别的 `/dev/serial/by-id/` 路径；若两板没有唯一序列号，则逐个接入识别。

以下只是示例，必须用本机实际路径替换：

```bash
export LEADER_PORT=/dev/ttyACM1
export FOLLOWER_PORT=/dev/ttyACM0
test -c "$LEADER_PORT" && test -r "$LEADER_PORT" && test -w "$LEADER_PORT"
test -c "$FOLLOWER_PORT" && test -r "$FOLLOWER_PORT" && test -w "$FOLLOWER_PORT"
```

也可运行官方 `lerobot-find-port`。其交互会要求拔插控制板；在 WSL 下重新插入后需再检查 USB/IP attach 状态。

## 6. 电机配置与校准

如果套件已由卖家配置好电机 ID 和波特率，不要直接重做 setup。仅对全新、未配置的电机执行，并严格按程序提示每次连接单个电机：

```bash
lerobot-setup-motors --teleop.type=so101_leader --teleop.port="$LEADER_PORT"
lerobot-setup-motors --robot.type=so101_follower --robot.port="$FOLLOWER_PORT"
```

然后分别校准两臂，依据交互提示设置中位并移动各关节至允许范围；不能强推机械限位：

```bash
lerobot-calibrate \
  --teleop.type=so101_leader \
  --teleop.port="$LEADER_PORT" \
  --teleop.id=so101_leader

lerobot-calibrate \
  --robot.type=so101_follower \
  --robot.port="$FOLLOWER_PORT" \
  --robot.id=so101_follower
```

后续持续使用相同 id，LeRobot 通过它们查找校准文件。默认存于 `~/.cache/huggingface/lerobot/calibration/` 下；环境变量或配置可以覆盖路径。

官方参考：[SO101 电机配置与校准说明](https://huggingface.co/docs/lerobot/so101)。

## 7. 第一次主从遥操作

先固定两臂、清空从臂运动范围，使两臂姿态大致对应，并让电机电源开关便于操作。先不加摄像头，以便验证串口和关节映射。

```bash
lerobot-teleoperate \
  --robot.type=so101_follower \
  --robot.port="$FOLLOWER_PORT" \
  --robot.id=so101_follower \
  --teleop.type=so101_leader \
  --teleop.port="$LEADER_PORT" \
  --teleop.id=so101_leader \
  --fps=30 \
  --display_data=false
```

缓慢移动主臂，观察从臂的关节方向和动作对应关系。Ctrl+C 退出。如果方向、行程或响应异常，停止并检查电机配置与校准，不能以“命令成功启动”作为实机验收。

通过标准：两臂能连接、六个关节映射正确、运动期间无连续串口超时、退出后行为符合预期。WSL 下实际控制频率与 USB 延迟仍需要现场验证。

官方参考：[LeRobot 遥操作说明](https://huggingface.co/docs/lerobot/il_robots)。

## 8. 后续摄像头与开发

基础跟随稳定后，再接摄像头并添加图像显示和数据记录。本机 Windows 能看到 HP 摄像头，但 Linux 没有 `/dev/video*`；WSLg 的显示能力不等于摄像头自动转接。摄像头需要单独验证 USB/IP、内核驱动、视频节点和实际帧率。

开始代码开发时，主要查看：

- `src/lerobot/teleoperators/so_leader/`：主臂读取。
- `src/lerobot/robots/so_follower/`：从臂控制。
- `src/lerobot/scripts/lerobot_teleoperate.py`：读取主臂动作、处理动作、发送从臂目标的循环。

当前方案是同一台电脑连接两臂。若未来需要跨网络遥操作，应额外设计连接中断后的停止行为、命令时效和关节限制，不能直接把本地串口循环视为完整网络控制方案。
