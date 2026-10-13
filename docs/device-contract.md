# 家庭设备工具契约

规范状态：`PROPOSED`。状态术语与维护人见[文档地图](index.md)。

本文是家庭设备工具的统一语义与后端适配边界的 Authority（权威位置）。系统职责见[软件架构](overview.md)；SimuHome 的运行与验证说明见[本地验证](simuhome.md)。

## 设计目标

Agent 面向稳定的家庭设备能力进行推理和调用。仿真平台与真实设备平台实现同一份上层契约，使运行循环、任务编排与实验中的目标判断可以复用。

职责分为三个边界：

| 边界 | 应承担的职责 |
| --- | --- |
| AgentRunner | 调用模型、分发已注册工具、接收结果、继续或结束运行 |
| 家庭设备工具 | 定义模型可见的参数和结果语义，按实际设备能力执行操作 |
| HomeBackend | 提供规范化的房间、设备、能力、状态与操作接口，完成平台标识和协议映射 |

SimuHomeBackend 是仿真验证后端。后续真实设备后端按同一契约接入，平台差异集中在适配器中。MCP 可以承载这些工具的发现与调用；设备语义由本文定义。

## 模型可见工具

这些工具是被测 Agent 在仿真实验中使用的家庭能力，覆盖设备发现、状态与环境读取，以及电源、亮度与灯光设置、温控、风速、窗帘、模式、目标温度、运行控制、洗衣选项、告警和电视播放。工具参数只使用规范单位和取值，不含平台端点、集群或原始编码。平台功能与这些工具的对应关系见[平台覆盖](#平台覆盖)。交付状态：`VERIFIED`（SimuHome 仿真范围）。实现位于运行代码仓库提交 `9848152`，单元测试、平台覆盖核对与真实模拟器核对见[证据覆盖](simuhome.md#证据覆盖)；真实设备后端尚未实现，不在此证据范围内。

| 工具 | 输入 | 应表达的结果 |
| --- | --- | --- |
| `get_rooms` | 无 | 可用房间及其真实标识 |
| `get_room_devices` | `room_id` | 该房间内的设备标识与设备类型 |
| `get_device_capabilities` | `device_id` | 设备具备的规范能力，以及每项能力的可读、可写与取值约束 |
| `get_device_state` | `device_id` | 全部可读能力的当前状态和证据来源 |
| `get_room_environment` | `room_id` | 房间温度（°C）、湿度（%）、光照（lx）、PM10（μg/m³），证据来源为 `simulated_environment_state` |
| `set_device_power` | `device_id`、`on: bool` | 电源 |
| `set_brightness` | `device_id`、`percent: 1–100` | 灯的亮度 |
| `set_power_on_brightness` | `device_id`、`percent: 1–100 或 null` | 以后每次开灯时的亮度；`null` 表示开灯时恢复关灯前的亮度 |
| `set_light_transition` | `device_id`、`seconds: 0–60`（最多一位小数） | 开关灯的渐变时间 |
| `set_thermostat_mode` | `device_id`、`mode` | 空调或热泵的运行模式 |
| `set_thermostat_setpoint` | `device_id`、`setpoint: cooling/heating`、`celsius` | 制冷或制热目标温度 |
| `set_fan_mode` | `device_id`、`mode` | 风扇类设备的档位 |
| `set_fan_percent` | `device_id`、`percent: 0–100` | 风扇类设备的风速百分比 |
| `set_cover_position` | `device_id`、`open_percent: 0–100` | 窗帘开合目标 |
| `stop_cover` | `device_id` | 让窗帘停在当前位置 |
| `set_device_mode` | `device_id`、`group`、`mode` | 设备在某一模式组中的模式 |
| `set_temperature_setpoint` | `device_id`、`celsius` | 冰箱、冷冻柜、洗衣机等设备的目标温度 |
| `control_operation` | `device_id`、`action: start/pause/resume/stop/go_home` | 洗碗机、洗衣机、烘干机或扫地机的运行 |
| `set_laundry_option` | `device_id`、`option: spin_speed/rinses/dryness`、`value` | 洗衣机转速、漂洗次数或烘干机烘干程度 |
| `set_alarm_enabled` | `device_id`、`alarm`、`enabled: bool` | 开启或关闭某类告警提醒 |
| `reset_alarms` | `device_id` | 复位设备已记录的告警 |
| `control_playback` | `device_id`、`action: play/pause/stop/start_over/next/previous/fast_forward/rewind` | 电视播放 |
| `set_tv_channel` | `device_id`、`major_number`、`minor_number` | 电视频道 |
| `step_tv_channel` | `device_id`、`direction: up/down` | 按频道表顺序切到下一个或上一个台，首尾循环 |

设置类工具的取值以 `get_device_capabilities` 返回的约束为准。设备未列出某项能力即表示不支持，相应设置返回明确错误，不以其他操作代替。

### 规范能力与状态

| 能力 | 能力描述中的约束 | 状态字段 |
| --- | --- | --- |
| `power` | `readable`、`writable` | 布尔值 |
| `brightness` | 百分比 1–100 | 百分比；设备尚未报告时为 `null` |
| `light_settings` | 开灯亮度 1–100 或 `null`、渐变 0–60 秒 | `power_on_brightness`（`null` 表示恢复关灯前亮度）、`transition_seconds` |
| `thermostat` | `modes`（`off`、`auto`、`cool`、`heat`）、设定范围 7–32 °C、制冷比制热至少高 0.25 °C | `mode`、`cooling_setpoint`、`heating_setpoint`、`current_temperature`（摄氏度） |
| `fan` | `modes`（按设备档位序列，取自 `off`、`low`、`medium`、`high`、`auto`）、百分比 0–100 | `mode`、`percent`；自动档时 `percent` 可为 `null` |
| `humidity` | 只读 | `measured_percent`；未报告时为 `null` |
| `cover` | 开合百分比 0–100，0 为全关、100 为全开；`stoppable` | `open_percent`、`target_open_percent`、`moving` |
| `modes` | `groups`：模式组到可选模式名称的映射，例如 `program`、`cooling_mode`、`run_mode`、`clean_mode` | 各模式组的当前模式名称 |
| `temperature_setpoint` | 设备报告的最小值、最大值与步长（摄氏度） | `setpoint`；设备有测温时附 `measured` |
| `operation` | 设备支持的 `actions` | `state`（`stopped`、`running`、`paused`、`error`，扫地机另有 `seeking_charger`、`charging`、`docked`）、`phase`、`countdown_seconds`；设备声明错误属性时附 `error`（如 `no_error`、`stuck`） |
| `laundry_options` | `options`：当前可选值；洗衣机转速的可选值随洗涤程序变化，运行中不可修改 | `spin_speed`、`rinses`、`dryness` 中设备具备的项 |
| `alarms` | 支持的告警种类 `alarms`（`inflow`、`drain`、`door`、`temp_too_low`、`temp_too_high`、`water_level`）、`resettable` | `active`（正在发生）、`latched`（发生过且未复位）、`enabled`（已开启提醒） |
| `playback` | 设备支持的 `actions` | `playing`、`paused`、`not_playing`、`buffering` |
| `channel` | 频道表 `channels`、`steppable` | `major_number`、`minor_number`、`name` |
| `electrical` | 只读，`fields` 列出设备具备的测量项 | `active_power_w`、`voltage_v`、`current_a`、`energy_imported_wh`；未报告的测量为 `null` |

可为 `null` 的状态只在上表注明的情形出现，表示设备没有报告该值，不代表默认值。其余状态缺失或类型不符时返回 `invalid_backend_response`，不推断默认状态。

部分设备要求先开机才接受其他设置，例如空调、空气净化器和电视；先后关系由设备反馈表达，调用方根据错误信息调整步骤。

### 统一结果字段

工具成功返回 `ok=true` 和规范化 `response`；失败返回 `ok=false` 与 `error`。后端错误同时保留稳定错误代码和实际反馈，平台拒绝原因附在错误消息中。后端提供的附加元数据和原生证据供来源追溯，决策使用规范能力与状态字段。

| 结果 | 规范字段 |
| --- | --- |
| 房间发现 | `rooms[]` 中的 `room_id` |
| 设备发现 | `room_id`、`devices[]` 中的 `device_id` 与 `device_type` |
| 能力描述 | `device_id`、`capabilities.<能力>`（`readable`、`writable` 及约束） |
| 当前状态 | `device_id`、`state.<能力>`、`evidence_source` |
| 设置结果 | `capability`、`requested`、`state.<能力>`、`confirmed`、单独的 `command` 应答证据 |
| 房间环境 | `room_id`、`environment`、`evidence_source` |

| 错误代码 | 含义 |
| --- | --- |
| `invalid_arguments` | 参数类型、范围或选项不符；未向设备发送任何操作 |
| `unsupported_capability` | 设备不具备该能力或不可写；未发送操作 |
| `ambiguous_<能力>_capability` | 设备有多个可对应的端点，不猜测控制目标 |
| `backend_request_failed` | 平台拒绝或请求失败，例如设备未开机、模式切换时设备正在运行 |
| `state_mismatch` | 命令已获应答，但回读状态不符合目标 |
| `state_confirmation_failed` | 命令已获应答，但状态回读失败 |
| `invalid_backend_response` | 平台返回的结构或状态缺失、类型不符 |

命令已获应答但回读失败或不匹配时，错误保留已执行命令及可用状态证据。调用方应据此查询或解释当前进度，不能从“未确认”推断命令没有执行。

窗帘等需要时间完成的动作，`confirmed` 表示目标位置已被设备接受，`moving` 表示仍在移动；是否到位以之后读取的 `open_percent` 为准。

`control_playback` 的 `start_over`、`next`、`previous`、`fast_forward`、`rewind` 在平台上没有可回读的状态变化，结果中 `confirmed` 为 `null`，只有命令应答可作证据。实验判定不以这些动作作为目标。

## 后端接口

`HomeBackend` 提供 `list_rooms`、`list_devices`、`get_device_capabilities`、`get_device_state`，以及与上表设置类工具同名的方法。家庭工具通过该接口访问设备平台；工具定义、模型上下文和运行循环使用统一语义。设置类方法统一先确认能力可写并校验参数，再发送操作并回读确认。

后端应处理以下映射：

- 标识：将上层房间与设备标识对应到平台中的实际对象。
- 能力：从平台发现结果生成规范化的能力描述。
- 操作：将统一动作转换为平台支持的命令或服务调用。
- 状态：将平台属性转换为统一状态与单位，并保留观测来源。
- 错误：保留操作失败和确认失败的事实，向上层提供可处理的信息。

SimuHomeBackend 按设备结构定位各能力所在端点与集群，在适配器内完成单位换算（温度 0.01 °C、亮度级别 1–254、窗帘关闭程度 0–10000）和枚举映射。枚举名称以固定上游提交的定义为准，因为原生写入整数后设备结构不再给出枚举表。后续 Home Assistant 后端需要映射真实实体标识、相应服务和上报状态；其实现与设备支持范围由真实接入实验验证。

## 平台覆盖

设备平台提供的每条命令、每个属性和每个房间环境量都应归入以下一类，归类结果连同理由记录在 `experiments/simuhome-coverage.json`，由 `experiments/check_coverage.py` 与设备目录逐项比对；任何一项未归类、引用不存在的工具，或某个工具没有对应的平台功能，核对即失败。

| 归类 | 含义 | 示例 |
| --- | --- | --- |
| 映射 | 由规范工具、状态字段或能力约束表达 | `OnOff.On` → `set_device_power` |
| 等价已覆盖 | 效果与已有工具相同，不另设工具，以免同义工具干扰 Agent 的选择和实验对比 | 亮度渐变命令 → `set_brightness` |
| 排除 | 写明理由不做：协议元数据或驱动细节；设备报告的事实或安装配置，开放写入会让被测 Agent 伪造事实；结果取决于执行前状态而无法核对 | `OnOff.Toggle`、窗帘动作计数器 |
| 含义不明 | 上游未声明语义，确认前不映射 | 电视的 LevelControl |

归类以 SimuHome 固定提交为准；上游版本变化时重新生成设备目录并重新核对。

## 执行与证据

工具应分别表达命令应答、设备状态和环境观测。状态确认只说明观测到的值与本次请求相符，其证据层次应明确：

| 证据 | 可以支持的结论 |
| --- | --- |
| 命令应答 | 后端受理或处理了命令 |
| 模拟设备状态 | 模拟器中的设备状态符合请求 |
| 真实设备上报状态 | 实体设备报告了相应状态 |
| 环境传感器观测 | 所观测的环境指标发生相应变化 |

SimuHome 回读标注为 `simulated_device_state`，房间环境标注为 `simulated_environment_state`。例如，灯的模拟开关值为 `true`，可以作为仿真任务的状态证据；实际照明效果由真实设备和环境观测提供证据。

## 实验平台的职责

实验入口准备场景、调用被测运行系统并独立核验结果。实验 Agent 按[名称目录约定](index.md#实验-agent-目录约定)组织，复用共享运行内核与统一工具实现；场景初始化、独立结果判定和观察界面由实验仓库管理。实验记录关联平台版本、Agent 名称、场景输入与被测源码版本。

本地验证覆盖上述统一能力在 SimuHome 后端的实现，核对方式见[本地验证](simuhome.md#核对扩展家庭能力)。真实后端接入后，对相同的能力语义和状态断言开展对应实验，并记录实际设备特有的延迟、离线和反馈情况。
