# 执行子 Agent 详细设计

规范状态：`PROPOSED` · 交付状态：`VERIFIED`（脚本化模型范围）。

证据覆盖：运行代码仓库提交 `09640d7` 的单元测试（121 项）与实验仓库提交 `aa45c88` 的测试（64 项）通过。脚本化模型在 4 个 SimuHome 实例上回放 `full`、`rule` 两种暴露策略与三种设备发现策略的 6 个组合，每组 C1–C8 × 20 变体共 160 次，全部有效且通过，首轮可见工具与本文定义一致；错房间、多余操作、虚报完成、歧义执行与先关后开等违规脚本均被判定器检出。`request` 与 `retrieve` 只有进程内测试，未经端到端回放；真实模型尚未运行。

本文是执行子 Agent 的任务输入、结构化结果、执行范围检查，以及设备发现与工具暴露候选策略的 Authority（权威位置）。它细化[软件架构](overview.md)中的[委派与反馈](overview.md#委派与反馈)和[动态暴露](overview.md#动态暴露)，不改变其中的角色职责。家庭设备工具语义以[统一工具契约](device-contract.md)为准，候选策略的比较方法见[工具暴露与设备发现对照实验](exposure-experiment.md)。

设计讨论来源：[home_agent-experiments#1](https://github.com/hongyue0721/home_agent-experiments/issues/1)。

## 范围

本文覆盖以下内容：

- 执行子 Agent 接收的任务；
- 它交回的结构化结果；
- Tool Executor 对执行范围的检查；
- 设备发现策略与工具暴露策略。

主 Agent、Task Runtime 的任务状态机、长期记忆和真实设备接入不在本文范围内。

尚无主 Agent 实现时，由实验程序扮演委派方，按本文格式构造任务并接收结果。执行子 Agent 复用 `runtime/` 中的 AgentRunner，不另建运行循环。

## 任务输入

委派方向执行子 Agent 提供以下信息。字段名为拟议名称，随实现确定。

| 字段 | 必需 | 含义 |
| --- | --- | --- |
| `task_id` | 是 | 任务标识，关联运行事件与结果 |
| `goal` | 是 | 自然语言目标，例如“把客厅的灯打开” |
| `scope` | 否 | 可执行范围：允许的房间、设备类型、能力或外部工具领域；缺省表示全屋家庭设备，不含外部工具 |
| `constraints` | 否 | 本次要求的附加约束，例如“不改变其他设备” |
| `context` | 否 | 完成任务所需的最小背景，不复制完整对话 |

任务输入不包含已解析的设备标识或期望状态。目标设备由执行子 Agent 通过发现工具定位；实验判定使用的期望状态只保存在用例文件中，不提供给被测 Agent。否则设备定位不再由被测对象完成，实验无法衡量发现能力。

`scope` 由 Tool Executor 强制执行，不只写在提示词中。同一用例在各对照组中使用相同的 `scope`。

## 结构化结果

执行子 Agent 通过 `report_result` 工具提交结果。提交后，本次运行结束。

| 字段 | 含义 |
| --- | --- |
| `status` | `completed`、`partial`、`failed`、`needs_clarification`、`unsupported` 或 `not_found` |
| `actions` | 子 Agent 声称执行的写操作，以工具调用标识引用 |
| `evidence` | 支持结论的回读状态或错误，以工具调用标识引用 |
| `unresolved` | 未完成的部分及原因 |
| `clarification` | `status` 为 `needs_clarification` 时必需：问题与候选对象 |

各状态的含义如下，同时写入系统提示与 `report_result` 的 `status` 参数说明：

| `status` | 含义 |
| --- | --- |
| `completed` | 全部目标达成，且有回读或确认证据 |
| `partial` | 至少一个目标达成且至少一个未达成 |
| `failed` | 尝试执行但目标未达成，例如设备拒绝或回读不符 |
| `needs_clarification` | 存在多个符合描述的候选对象，无法确定执行哪个，未执行写操作 |
| `unsupported` | 目标设备存在，但不具备所需能力 |
| `not_found` | 任务指定的房间或设备不存在 |

`report_result` 的内容是子 Agent 的自述，不作为完成证据。实验判定会把自述与平台原生状态和运行事件对照，判断是否存在虚假完成。

运行若以普通文本回复结束、未调用 `report_result`，或达到轮数上限，均记录为“未提交结构化结果”。这种情况不由 Runtime 代为补出结果。

## 执行范围检查

Tool Executor 在调用后端之前，检查写操作是否属于 `scope`。`scope` 约束家庭设备的写操作与全部外部工具调用；家庭设备的读取和发现不受限制，以便子 Agent 能发现歧义或确认目标不存在。

写操作同时满足以下条件才属于范围：

- `scope.rooms` 存在时，目标设备所在房间在其中；
- `scope.device_types` 存在时，目标设备类型在其中；
- `scope.capabilities` 存在时，该工具对应的能力在其中。

设备所在房间与类型由 Runtime 在运行开始时从后端读取，不取自模型参数。

外部工具（非家庭工具，见[外部工具](#外部工具)）的调用，不论只读还是写入，都只在 `scope.domains` 列出其领域时属于范围；`scope` 或 `scope.domains` 缺省时，外部工具均不属于范围。只读的外部调用同样受限，是因为在真实部署中查询会把任务信息发送到家庭之外的服务。

- **超出范围**：不调用后端，向模型返回错误码 `out_of_scope`，同时在运行事件中记录这次越界尝试。
- **工具未暴露**：调用目录中存在、但本轮不可见的工具时，返回 `tool_not_exposed` 并记录。目录中不存在的工具名仍返回 `unknown_tool`。
- **未知设备**：写操作的目标设备不在运行开始时读取的设备清单中，无法确认其房间与类型，不调用后端，返回 `unknown_device`。编造的设备标识与越界尝试分开计数。

可见性与授权相互独立：工具暴露只决定模型能看到什么，不扩大可执行范围；可执行范围也不随暴露策略变化。

`out_of_scope`、`tool_not_exposed`、`unknown_device` 和 `not_executed` 属于运行时错误，不是家庭设备后端错误。实现时需在[统一工具契约](device-contract.md)的错误码表中登记，或单独说明其归属。

## 工具分类

暴露策略只作用于**操作工具**。以下三类工具在所有策略下始终可见：

- 发现工具（由所选设备发现策略决定）；
- `get_device_capabilities`、`get_device_state`、`get_room_environment`；
- `report_result`。

操作工具指会改变状态的工具，包括统一契约中以 `set_`、`control_`、`step_`、`stop_`、`reset_` 开头的工具，以及接入的非家庭工具（见[工具目录规模](exposure-experiment.md#工具目录规模)）。

工具目录中的每个条目记录：来源、领域（如 `home`、`web`、`calendar`）、对应能力、是否写操作。规则筛选和目录检索依据这些元数据，不解析提示词。

## 设备发现策略

三种策略提供的信息相同：设备标识、所在房间和设备类型。区别只在于模型获取信息的方式。能力与状态在三种策略下都通过 `get_device_capabilities` 与 `get_device_state` 查询。

| 策略 | 标识 | 模型获取设备信息的方式 |
| --- | --- | --- |
| 渐进发现 | `progressive` | 现有工具：`get_rooms` → `get_room_devices`，逐层查询 |
| 扁平工具 | `flat_tool` | 新增 `list_all_devices`，一次返回全屋设备及所在房间；不提供 `get_rooms` 与 `get_room_devices` |
| 扁平清单 | `flat_prompt` | 运行开始时由 Runtime 把全屋设备清单写入任务上下文；不提供上述发现工具 |

设备元数据只来自平台或场景文件，不从设备标识推断。SimuHome 只提供房间和设备类型。楼层、别名和标签由实验场景补充：参照 Home Assistant 的区域（area）、楼层（floor）、别名（alias）、标签（label）建模，标注为测试设定，并在三种策略下以相同方式出现在发现结果中。

## 工具暴露策略

四种策略的最终可执行能力相同，只改变模型在各轮请求中可见的操作工具。

| 策略 | 标识 | 可见的操作工具 |
| --- | --- | --- |
| 全量暴露 | `full` | 运行开始即可见目录中全部操作工具 |
| 规则筛选 | `rule` | 运行开始前，Tool Selector 按 `scope.capabilities`（家庭工具）与 `scope.domains`（外部工具）确定性地筛选；运行中不变 |
| 模型申请 | `request` | 初始不可见；模型调用 `request_tools` 检索目录，Runtime 把匹配工具的定义加入下一轮请求 |
| 语义检索 | `retrieve` | 运行开始前，Runtime 以任务 `goal` 的向量检索操作工具，取相似度最高的 k 个；运行中不变 |

暴露的触发条件不依赖设备发现结果。若按“发现了哪些设备能力就暴露哪些工具”触发，在扁平发现下全屋能力一次可见，动态暴露会退化为全量暴露，两个维度无法分离。

规则筛选要求用例的 `scope` 声明能力。缺少声明时，该用例不参与规则筛选组，不回退为全量暴露。

`request_tools` 采用确定性的关键词与元数据匹配，返回工具名、领域和简短说明。已加入的工具在本次任务内保持可见，任务结束后释放。

`retrieve` 使用嵌入模型服务，不引入本地向量库依赖。各工具的检索文本为工具名、能力、领域与描述的拼接，向量按“工具目录哈希 + 嵌入模型名”缓存在调用方指定的目录中，目录不变时不重复计算；缓存文件校验失败时报错，不静默覆盖。检索文本按行拼接 `name`、`capability`（为 `null` 时省略该行）、`domain`、`description`；缓存文件名为模型名与检索文本的 SHA-256，内容含 `model`、`key`、`dims` 与 `vectors`（工具名到向量）。相似度为余弦相似度，同分时按目录顺序；零向量无法计算余弦，按嵌入失败处理。嵌入失败时运行失败，不回退为全量暴露。`retrieve` 下 `request_tools` 不可见。k 为显式参数，没有缺省值，并记录在每次运行中。检索只看任务文本，不读取设备发现结果。嵌入请求的 Token 与用时单独记录，计入该组成本。

与 `retrieve` 相近的现有工作中，RAG-MCP 在工具选择前，先用语义检索从大量 MCP 服务描述中挑出候选，再交给模型。它报告的工具选择准确率高于把全部描述放入提示词的做法（[arXiv:2505.03275](https://arxiv.org/abs/2505.03275)）。它的任务与规模和本项目不同，结论不直接适用。

## 实现接口

本节是 `runtime/` 与 `experiments/` 之间的接口，已在运行代码仓库提交 `09640d7` 与实验仓库提交 `aa45c88` 实现。现有 `--task` 单执行 Agent 入口保持原行为，不受影响。

### 任务文件

`python -m banju` 新增 `--task-file <path>`，与 `--task` 互斥。文件为 JSON 对象：

```json
{
  "task_id": "C3-v07-r2",
  "goal": "把客厅的灯打开",
  "scope": {"rooms": ["living_room"], "device_types": null, "capabilities": ["power"]},
  "constraints": ["不改变其他设备"],
  "context": null
}
```

- `task_id`、`goal` 为非空字符串；`scope`、`constraints`、`context` 可省略或为 `null`。
- `scope` 的三个字段均为非空字符串列表或 `null`。`capabilities` 取规范能力名，即 `get_device_capabilities` 返回的能力键。
- 字段不合法时，入口以配置错误退出，不启动运行。

### 策略参数

使用 `--task-file` 时必须同时给出以下两个参数，没有缺省值：

| 参数 | 取值 |
| --- | --- |
| `--discovery` | `progressive`、`flat_tool`、`flat_prompt` |
| `--exposure` | `full`、`rule`、`request`、`retrieve` |

`--exposure rule` 要求任务的 `scope.capabilities` 非空，否则以配置错误退出。

`--exposure retrieve` 另需 `--retrieve-top-k`（正整数）、`--embedding-base-url`、`--embedding-model`、`--embedding-api-key-env` 与 `--embedding-cache-dir`（已存在的目录）；缺少任一项即配置错误。其他暴露策略给出这些参数，同样是配置错误。嵌入服务按 OpenAI-compatible `/embeddings` 接口调用。

### 限流重试

`--rate-limit-max-wait <秒>` 为可选参数，`--task` 与 `--task-file` 两个入口均支持，同时作用于模型与嵌入请求。未提供时不重试，行为与原来相同。

提供时，只有 HTTP 429 会重试：429 表示请求未被处理，重试不改变模型行为。等待时间优先取响应头 `Retry-After` 中的秒数；该头缺失或为日期等其他形式时，采用指数退避，上限依次为 2、4、8……60 秒，在 0 到当前上限之间均匀取值。等待预算按每次模型或嵌入调用分别计算，下一次等待将使累计等待超过预算时停止重试，仍以 `service` 类错误结束，错误信息写明重试次数与累计等待。超时、传输错误和其他 HTTP 错误不重试。

模型记录增加 `rate_limit_retries` 与 `rate_limit_wait_s`；嵌入记录的这两个字段为 `{"tools": ..., "goal": ...}`，工具向量命中缓存时 `tools` 为 `null`。记录的起止时间覆盖等待，统计模型延迟时应扣除 `rate_limit_wait_s`。`run_started` 记录 `rate_limit_max_wait`，未提供时为 `null`。

### 工具目录元数据

每个工具条目包含：

- `definition`：发给模型的工具定义；
- `kind`：`discovery`、`read`、`operation` 或 `control`（`report_result` 与 `request_tools`）；
- `capability`：该工具对应的规范能力名，非设备工具为 `null`；
- `domain`：工具领域，家庭工具为 `home`。

外部工具另含 `source`（服务标识与版本）与 `effect`（`read` 或 `write`），`kind` 为 `operation`，`capability` 为 `null`。

### 外部工具

本节接口已在运行代码仓库提交 `a8db23e` 实现（单元测试 179 项通过；尚未与实验仓库端到端联调），用于[工具目录规模](exposure-experiment.md#工具目录规模)的异构扩充，`--task` 入口不受影响。

- `--external-tools <文件>`：外部工具目录。每个条目含工具定义（名称已加服务前缀）、`domain`、`effect`、`source` 与默认应答 `default_response`。条目追加在家庭工具之后，顺序即目录顺序；与已有工具重名、缺少默认应答或字段不合法时为配置错误。
- `--tool-fixtures <文件>`：可选，工具名到应答的映射，覆盖对应工具的默认应答；出现目录中不存在的工具名时为配置错误。
- **执行**：外部工具通过可见性与范围检查后，不访问网络，返回 `{"ok": true, "response": <应答>}`；写入类工具同样只返回应答，并在运行事件中如实记录调用。
- **参数校验**：不引入 JSON Schema 依赖，只校验参数为对象、必填字段齐全、`additionalProperties` 为 `false` 时没有多余字段，以及顶层字段的基本类型；不通过时返回 `invalid_arguments`。
- **暴露**：`full` 包含全部外部工具；`rule` 下外部工具在其 `domain` 属于 `scope.domains` 时可见；`request_tools` 与 `retrieve` 对外部工具使用与家庭工具相同的匹配与检索文本。
- **记录**：`run_started` 记录外部目录与固定应答文件的 SHA-256。

### 工具申请 request_tools

参数为 `query`（非空字符串）。

匹配规则：把 `query` 按空白和中英文逗号切分为若干词；词中连续的中文（U+4E00–U+9FFF）片段再切成相邻两字的片段，单字片段丢弃，其余片段保持原样。各片段与操作工具的 `name`、`capability`、`domain` 和描述做不区分大小写的子串匹配，任一片段命中即算匹配，结果按目录顺序返回。中文查询通常不含空格，整串匹配几乎无法命中；单字（如“设”“的”）又会命中大多数工具，因此采用两字片段。

返回匹配工具的名称、领域、能力和描述。匹配到的工具从下一轮模型请求起可见，在本次任务内保持可见。未匹配时返回空列表，不作为错误。

### 结果提交 report_result

参数与[结构化结果](#结构化结果)一致：

- `status` 为枚举；
- `actions`、`evidence` 为工具调用标识的字符串列表；
- `unresolved` 为字符串列表；
- `clarification` 为对象 `{question, candidates}` 或 `null`，`status` 为 `needs_clarification` 时必须非空。

参数校验失败时，按普通工具错误返回给模型，运行继续。校验通过时记录结果并结束运行，停止原因为 `report_result`。同一条模型消息中排在合法报告之后的工具调用不执行，结果记为 `not_executed`，仍写入记录。

`partial` 表示至少一个目标已达成，同时至少一个目标未达成。

### 运行结果与事件

在现有格式之上追加：

- `run_started` 增加 `task`（任务对象）、`discovery`、`exposure`。
- 每轮请求前，可见工具集合有变化时，发出 `tools_exposed` 事件，内容为 `step` 与 `tools`（工具名列表）；首轮必发。
- 模型记录增加 `tools`：该轮实际发送的工具名列表。
- 运行结果增加 `report`：`report_result` 的参数；未提交时为 `null`。
- `retrieve` 下，`run_started` 另含 `retrieve_top_k` 与 `embedding_model`；第一条模型记录之前增加一条 `kind` 为 `embedding` 的记录，含起止时间、`cache_hit`、`usage`（`tools` 为工具向量请求、`goal` 为任务向量请求；缓存命中或提供方未报告时为 `null`）、选中工具及其相似度 `selected`，以及实际选中数量 `selected_count`（k 大于操作工具数时小于 k）。

### 错误码

以下运行时错误与现有工具错误同形，即 `{"ok": false, "error": {"type": ..., "message": ...}}`：

| `type` | 含义 |
| --- | --- |
| `out_of_scope` | 写操作或外部工具调用超出任务范围，未执行 |
| `tool_not_exposed` | 调用目录中存在、但本轮不可见的工具，未执行 |
| `unknown_device` | 写操作目标不在运行开始时的设备清单中，未调用后端 |
| `not_executed` | 合法 `report_result` 之后的同条消息调用，运行已结束，未执行 |

运行因异常结束时，结果的 `error` 对象在来源已知时增加 `category` 字段，`type` 与 `message` 不变：

| `category` | 含义 |
| --- | --- |
| `service` | 模型或嵌入服务的 HTTP 错误、传输错误、超时或响应不符合协议 |
| `output` | 模型返回的消息或工具调用不合法 |
| `infrastructure` | 运行设施自身的故障，如嵌入向量缓存文件损坏 |

## 实现影响

以下改动已在运行代码仓库提交 `09640d7` 实现：

- **AgentRunner**：支持每轮请求使用不同的工具集合；以 `report_result` 作为结束条件。
- **工具目录**：
  - 为条目增加元数据；
  - 实现 Tool Selector 的四种暴露策略、`request_tools` 与嵌入检索；
  - 实现三种设备发现策略。
- **Tool Executor**：增加范围检查和未暴露工具检查。
- **统一工具契约**：新增 `list_all_devices`，返回全部设备的 `device_id`、`device_type` 与 `room_id`；登记运行时错误码。

## 已确定的设计选择

以下选择由设计确定（2026-10-13），随实验结果复核：

- `status` 采用六种取值，`partial` 的边界见 [结果提交](#结果提交-report_result)。
- `scope` 允许按房间、设备类型和能力声明，三者均可选。外部工具接入后增加可选的 `domains`，只约束外部工具（维护人确认，2026-10-14）；家庭工具的领域 `home` 不需声明。
- 楼层、别名等场景元数据不在第一阶段引入，只在设备规模实验中引入。
