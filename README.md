# CMD

控制命令中枢：汇总遥控器与上位机输入，发布底盘、云台与发射命令 / Control command hub that merges remote-controller and host inputs and publishes chassis, gimbal and launcher commands

本仓库派生自 [QDU-Robomaster/CMD](https://github.com/QDU-Robomaster/CMD)（Apache-2.0），改动见 `NOTICE`。

This repository is derived from [QDU-Robomaster/CMD](https://github.com/QDU-Robomaster/CMD) (Apache-2.0); see `NOTICE` for the changes.

## 1. 模块作用 / Purpose

CMD 是整台车的命令中枢，本身不碰硬件。它把两个来源的数据汇总，按当前控制模式发布三路命令 Topic：

- 遥控器：`FeedRC()` 写入。数据来自 Fsi6x 一类遥控解析模块。
- 上位机 / 自动控制：`FeedAI()` 写入。数据来自自瞄一类模块。

遥控器侧有多个输入槽（`RCInputSource`），CMD 按下面的规则挑一路：

- `RC_INPUT_FSI6X`（Fsi6x 遥控接收机）与 `RC_INPUT_VT13`（图传链路遥控）各占一个槽，数据互相独立。
- 当前活动槽只要在线（`chassis_online`）就继续用；活动槽掉线后切到另一路在线的槽；两路都掉线则输出全零的离线数据。
- 切换是**粘性**的：从活动槽切走之后，除非新槽掉线，否则不会自动切回。

控制模式：

- `CMD_OP_CTRL`（操作手控制）：遥控器数据原样发布到三路 Topic。
- `CMD_AUTO_CTRL`（自动控制）：底盘、云台分别取"AI 在线则用 AI，否则用遥控器"；开火取 AI 与遥控器的**逻辑与**，两边都按下才开火。

遥控器上线 / 离线时，CMD 各自激活一次 `CMD_EVENT_START_CTRL` / `CMD_EVENT_LOST_CTRL`。另外 `GetEvent()` 上注册了 `Mode` 的数值作为事件 ID，激活该数值即切换控制模式 —— EventBinder 就是靠这一点把拨杆事件接到模式切换上的。

CMD is the command hub of the robot and touches no hardware itself. It merges two sources and publishes three command Topics according to the current control mode:

- Remote controller, written by `FeedRC()`, fed by a remote-controller parsing Module such as Fsi6x.
- Host / automatic control, written by `FeedAI()`, fed by an auto-aim Module.

The remote-controller side has several input slots (`RCInputSource`) and CMD picks one:

- `RC_INPUT_FSI6X` (Fsi6x receiver) and `RC_INPUT_VT13` (video-link remote controller) each own a slot, and their data are independent.
- The active slot is kept as long as it is online (`chassis_online`); once it goes offline CMD switches to the other online slot, and with both offline it outputs all-zero offline data.
- The switch is **sticky**: after moving away from the active slot, CMD does not move back until the new slot goes offline.

Control modes:

- `CMD_OP_CTRL` (operator control): the remote-controller data is published to the three Topics unchanged.
- `CMD_AUTO_CTRL` (automatic control): chassis and gimbal each take the AI value when the AI is online and the remote value otherwise; firing is the **logical AND** of the AI and remote values, so both must be held.

CMD activates `CMD_EVENT_START_CTRL` / `CMD_EVENT_LOST_CTRL` once when the remote controller comes online and goes offline. The numeric values of `Mode` are also registered as event IDs on `GetEvent()`, so activating one switches the control mode — this is what EventBinder uses to wire a switch to a mode change.

## 2. 构造接口 / Constructor

```cpp
CMD(CMD::Mode mode = CMD::Mode::CMD_OP_CTRL,
    const char* chassis_cmd_topic_name = "chassis_cmd",
    const char* gimbal_cmd_topic_name = "gimbal_cmd",
    const char* launcher_cmd_topic_name = "launcher_cmd");
```

配置参数 / Configuration parameters：

- `mode`：初始控制模式，默认操作手控制。
- 三个 Topic 名：留空时用默认名。

## 3. Topic

发布 / Published：

| 名称 | 类型 | 用途 |
| --- | --- | --- |
| `chassis_cmd` | `CMD::ChassisCMD` | 底盘 `x` / `y` / `z` 与自定义模式 |
| `gimbal_cmd` | `CMD::GimbalCMD` | 云台 `yaw` / `pit` / `rol` 及其一二阶导 |
| `launcher_cmd` | `CMD::LauncherCMD` | 发射 `isfire` |

订阅 / Subscribed：无 / None

Topic 名可在构造时改写。底盘、云台、发射模块订阅这三路即可，不需要直接引用 CMD。

The Topic names can be overridden in the constructor. Chassis, gimbal and launcher Modules only need to subscribe to these three; they do not reference CMD directly.

## 4. 配置示例 / Configuration Example

CMD 没有构造依赖，直接列出即可。它必须排在所有调用 `FeedRC` / `FeedAI` 的模块**之前**：

CMD has no constructor dependencies and can simply be listed. It must appear **before** every Module that calls `FeedRC` / `FeedAI`:

```yaml
modules:
  - module: starpang888/CMD
    id: cmd
    args:
      - mode: CMD::Mode::CMD_OP_CTRL
  - module: starpang888/Fsi6x
    id: fsi6x
    args:
      - uart: usart3
      - cmd: cmd
```

把拨杆接到模式切换上，用 EventBinder：

Wire a switch to the mode change with EventBinder:

```yaml
  - module: starpang888/EventBinder
    id: event_binder
    args:
      - modules: '{{"fsi6x", fsi6x}, {"cmd", cmd}}'
      - event_binding_groups: '{{{{"fsi6x", Fsi6x::SwitchPos::FSI6X_SW_L_POS_BOT, "cmd", CMD::Mode::CMD_AUTO_CTRL}}}}'
```

## 5. 依赖与硬件 / Dependencies and Hardware

模块依赖 / Module dependencies：

无 / None

硬件 / Hardware：

无 / None
