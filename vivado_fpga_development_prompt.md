# AMD Vivado FPGA 项目开发规范

## 1. 适用范围

本规范适用于所有 AMD/Xilinx FPGA 相关任务，包括：

- Verilog/SystemVerilog RTL 编写、修改和审查；
- Vivado IP、Block Design、Processing System、时钟和复位配置；
- RTL lint、elaboration、综合、实现和时序分析；
- XSim、Questa、ModelSim 等 RTL 仿真；
- ILA、VIO 和 Hardware Manager 调试；
- Vivado 工程脚本化、版本控制和可复现构建；
- FIFO、CDC、AXI、GMII、LVDS、GT/PHY 和报文处理链路。

## 2. AMD Vivado MCP 和 Skill 优先级

涉及 Vivado 或 FPGA 开发时，先检查当前环境中是否存在可用的 AMD Vivado MCP 和对应 Skill。

优先使用：

- Vivado MCP 的工程、IP、综合、实现、硬件管理和文档查询能力；
- `vivado-ip-configurator`；
- `vivado-rtl-lint`；
- `vivado-rtl-elaboration-analysis`；
- `vivado-simulate-rtl`；
- `vivado-timing-methodology-checks`；
- `hw-ila-debug`；
- `hw-vio-debug`；
- `vivado-revision-control`；
- 项目提供的 Verilog/FPGA 风格 Skill。

任务路由如下：

| 任务 | 优先 Skill |
|---|---|
| 配置或修改 Vivado IP | `vivado-ip-configurator` |
| RTL 静态检查 | `vivado-rtl-lint` |
| elaboration、综合报错分析 | `vivado-rtl-elaboration-analysis` |
| RTL、门级或第三方仿真 | `vivado-simulate-rtl` |
| TIMING-*、UG906 或方法学违规 | `vivado-timing-methodology-checks` |
| ILA 发现、触发、采集和导出 | `hw-ila-debug` |
| VIO 读写、驱动和监控 | `hw-vio-debug` |
| 工程可复现、版本控制和构建脚本 | `vivado-revision-control` |

如果目标 MCP 或 Skill 当前不可用，必须明确说明实际不可用的能力，然后使用项目已有的 Tcl、Batch、Makefile、仿真脚本或 Vivado 工具链作为后备方案。不得声称已经调用不可用的 MCP。

### 2.1 Prompt、Agent、Skill 与 MCP 的分工

这份文件是项目级 Prompt（提示词）和执行规约，不是一个 MCP Server，也不是某个客户端的模型本身。为了避免把配置、流程和执行能力混为一谈，按下面的边界理解：

| 层次 | 负责什么 | 在本项目中的例子 |
|---|---|---|
| Prompt | 固定目标、约束、权限边界、确认门槛和验收证据；回答“应该怎样做” | 本文件规定先只读分析、保持接口稳定、仿真前必须获授权 |
| Agent | 读取上下文并规划步骤，决定何时使用 Skill 或调用 MCP，汇总结果；回答“下一步做什么” | Claude Code、Codex 或其他支持工具调用的客户端中的模型循环 |
| Skill | 可复用的领域工作流、模板、参数和检查表；回答“按什么流程做” | `vivado-rtl-lint`、`vivado-simulate-rtl`、`hw-ila-debug` |
| MCP | 为客户端提供外部 Tools、Resources 和 Prompts 原语；真正连接 Vivado、文档或硬件 | AMD Vivado MCP、AMD 文档搜索 MCP |

一次典型调用链是：

```text
用户任务
  -> Agent 读取本 Prompt
  -> Agent 选择匹配的 Skill
  -> MCP Client 调用 MCP Server 的工具或资源
  -> 返回日志/报告/数据
  -> Agent 按本 Prompt 汇总证据并报告结果
```

MCP 规范中的 `prompts` 原语与配置管理器里的“Prompt 预设”不是同一个概念：前者是 MCP Server 暴露给 Host 的模板能力，后者通常是写入 `CLAUDE.md`、`AGENTS.md` 等客户端提示词文件的文本。两者都可以存在，但不能因为配置界面里都叫 Prompt，就假设它们具有相同的作用。

### 2.2 用 CC Switch 管理多客户端配置（可选）

[CC Switch](https://github.com/farion1231/cc-switch) 是第三方本地桌面配置管理器，不是 AMD 官方组件、模型运行时或 Vivado MCP Server。它适合解决“同一套 Prompt、Skill 和 MCP 配置在多个客户端之间漂移”的问题：集中导入、备份、启用和同步，实际执行仍由目标客户端的 Agent 和 MCP Client 完成。版本和支持的客户端会变化，使用前以[官方 Releases](https://github.com/farion1231/cc-switch/releases)和手册为准。

建议按以下原则使用：

1. 将本文件保留在项目仓库，作为唯一规则源；在 CC Switch 中导入副本时，先比较差异，并记录启用的客户端和版本。
2. Prompt 预设通常会写入目标客户端的系统提示词文件，例如 Claude 的 `~/.claude/CLAUDE.md`、Codex 的 `~/.codex/AGENTS.md`。同一客户端启用新预设可能覆盖旧文件，启用前先备份或导出。
3. MCP 面板只负责登记和同步 Server 配置，不保证 Server 已启动、端口可达、证书有效或权限足够。同步后仍需在客户端中做只读连通性检查。
4. Skill 必须是可审查的目录和 `SKILL.md`（或目标客户端规定的等价结构），不能把一段普通 Prompt 文本误当作 Skill。来自 GitHub、ZIP 或第三方源的 Skill/MCP 先审查脚本、命令、参数和环境变量。
5. API key、OAuth 凭据和本地配置目录不得写入本文件、文章或 Git。CC Switch 的本地数据目录可能包含凭据和备份，禁止上传或提交。

CC Switch 只是配置控制台；“切换供应商/模型”“启用 Prompt”“同步 Skill”“登记 MCP”是四个不同动作，不能把其中任何一个动作宣传成已经完成 FPGA 分析、仿真或硬件操作。

### 2.3 可复用任务 Prompt 的最小结构

新增任务 Prompt 时，至少明确以下字段，避免只写一句“帮我修改 RTL”：

```text
目标：要解决的工程问题和期望产物
上下文：工程路径、器件、Vivado 版本、模块链路、相关日志
约束：接口、编码、时钟/复位、不可改变的文件和安全边界
允许的工具：可用 Skill、MCP、脚本；不可用时的后备方案
执行阶段：只读检查 -> 复述约束 -> 用户确认 -> 修改 -> 检查
验收证据：编译/报告/断言/计数/内容比对/波形/文件差异
暂停条件：协议冲突、接口变化、覆盖文件、仿真或硬件操作等
交付物：修改文件、命令、日志位置、未覆盖项和下一步建议
```

这份项目规范已经把上述字段扩展成 RTL、Vivado、仿真、时序、硬件调试和交接记录的具体规则。使用时优先维护本文件，再将必要的短版规则同步给具体客户端。

## 3. 项目信息卡

开始任务时先建立或读取以下项目资料：

| 项目 | 内容 |
|---|---|
| 项目名称 | `[项目名称]` |
| 工程根目录 | `[绝对路径]` |
| 目标器件 | `[FPGA 器件型号]` |
| Vivado 版本 | `[Vivado 版本]` |
| 主要 RTL 目录 | `[RTL 目录]` |
| 仿真目录 | `[仿真目录]` |
| 约束目录 | `[XDC 目录]` |
| 关键模块链路 | `[模块 A] -> [模块 B] -> [模块 C]` |
| 时钟和复位 | `[时钟频率、复位极性、复位域]` |
| 数据接口 | `[AXI/LVDS/GMII/GT/自定义接口]` |
| 编码约束 | `[原文件编码或项目编码要求]` |
| 参考资料 | `[协议、数据手册、已有 RTL、IP 网表]` |

## 4. 开始任务时的只读分析

开始任务后，先进行只读检查：

- 当前 RTL 和顶层模块；
- 所有相关调用点和实例化关系；
- Vivado 工程、Block Design、IP 配置和 fileset；
- XDC 时钟、引脚、IO、时序和 CDC 约束；
- 仿真脚本、测试平台、模型和库；
- 现有综合、elaboration、仿真和实现日志；
- 项目文档和 `docs_<agent-name>/progress.md` 进度记录；
- 当前 Git 状态和已有用户修改。

然后用简短文字复述：

- 应用场景和数据流向；
- 模块链路；
- 帧格式、长度、字段位置和字节顺序；
- 时钟、复位、FIFO、CDC 和接口握手要求；
- 当前代码中与任务直接相关的状态机和边界条件；
- 计划修改的文件和验证范围；
- 需要用户确认的协议或接口假设。

只读分析可以直接执行。修改 RTL、生成新的仿真激励、重新生成 IP、启动仿真、综合或实现前，必须先完成约束复述。

## 5. 需要暂停并向用户确认的情况

遇到以下情况时暂停并请求确认：

- 协议、帧长度、字段位置或字节顺序存在冲突；
- 需要改变公共模块接口；
- 需要删除、覆盖或批量重写用户文件；
- 需要重新生成或升级 IP；
- 仿真模型、IP 网表、器件库或约束缺失；
- 测试结果与协议预期不一致；
- 需要对 TIMING 方法学违规执行 USER 类修复；
- 需要给硬件重新下载 bitstream/PDI；
- 需要保存或保留 Vivado 生成目录；
- 需要启动仿真、综合、实现或硬件采集等可能耗时的操作。

仿真或 Vivado 长时间任务开始前，应报告准备执行的命令或 MCP 操作、使用的工程和工具版本、验证范围、预期结果以及日志和波形保存位置。只有用户明确授权后才启动仿真或硬件采集。

## 6. RTL 修改约束

- 默认保持公共接口稳定。
- 如果必须改接口，先列出旧接口、新接口和全部调用点。
- 明确 `req/ack/done`、`valid/ready`、`sop/eop` 的时序关系和保持周期。
- 报文级请求需要锁存一组并行字段时，明确锁存时刻、完成时刻和异常退出条件。
- FIFO 以 `valid`、`dout_vld` 或对应 IP 明确给出的数据有效信号为准。
- 不根据猜测的固定拍数判断 FIFO 数据已经有效。
- 异步 FIFO 必须明确两侧时钟、复位域、CDC 和反压策略。
- 明确计数器的起始值、结束值、越界条件和最大帧长行为。
- 状态机必须说明每个状态的进入、退出和异常恢复条件。
- 每个端口单独一行，每条连续赋值单独一行。
- 注释保持项目原有语言和编码风格。
- 不以“编译通过”代替功能正确结论。
- 功能结论必须由断言、计数、内容比对、报告或波形证据支持。

## 7. RTL 编码和文件编码

修改 `.v`、`.sv`、`.vh`、`.svh` 文件前，先检测原文件编码。

- 默认保持原文件编码；
- 项目明确要求 GB2312 时，修改后执行 GB2312 往返校验；
- 不要把无关 RTL 文件统一转换成 UTF-8；
- Markdown、TCL、Batch、PowerShell 和普通文档按原有编码保持；
- 如果编码无法确认，先报告检测结果和风险。

## 8. Vivado MCP 使用规则

### 8.1 Vivado 会话

使用 Vivado MCP 时：

- 先用 `vivado_list_sessions` 查找会话；
- 使用 `vivado_status` 检查会话健康状态；
- 每次 `vivado_execute` 都传入明确的 `session_id`；
- 复用硬件会话前，检查 Vivado 版本与 `hw_server` 版本；
- 发现版本不匹配时，不自动重试或重启，报告两个实际版本；
- 需要时使用 `vivado_doc_search` 查询陌生 Tcl 命令、属性或参数。

### 8.2 Tcl 执行

- 尽量批量执行相关 Tcl，减少重复 MCP 调用；
- 不重复尝试未知语法；
- Tcl 报错后先查询对应 Vivado 文档或 Skill 参考资料；
- 确认语法和对象后再进行下一次调用；
- 将大型报告、波形或采样数据写入文件，聊天中只给摘要；
- 不把完整日志或波形数据直接粘贴到对话中。

### 8.3 硬件调试

对于 ILA，按“发现 ILA 和 probe、核对 bitstream 与 `.ltx`、配置触发、arm、等待触发、upload、导出、分析”的顺序执行。

- 优先使用 Vivado MCP；
- 没有 ILA 时停止并说明 bitstream 或 `.ltx` 问题；
- 不使用 `display_hw_ila_data` 打开 GUI；
- 使用 `write_hw_ila_data`、`list_hw_samples` 等方式导出数据；
- 可能不触发的条件先考虑 `run_hw_ila -trigger_now`；
- `wait_on_hw_ila -timeout` 的单位是分钟；
- Versal 上由 VIO 触发 ILA 时，arm 和触发操作使用分开的 MCP 调用；
- 每次成功采集都生成 `capture_data.csv`、`report_data.json` 和 `REPORT.md`；
- 报告中使用实际 probe 名称、宽度、采样位置和数值。

对于 VIO，按“发现 VIO 和 probe、刷新输入值、读取实际输入值、同步旧输出值、设置输出、`commit_hw_vio`、记录前后值”的顺序执行。

重新下载 bitstream 后，先刷新 VIO 输出值，避免使用旧会话残留值。任何需要重新下载 PDI 或 bitstream 的操作，都必须先向用户确认。

## 9. IP 和 Block Design

配置任意 Vivado IP 时优先使用 `vivado-ip-configurator`：

- 先确认 IP VLNV、版本、器件和接口；
- 从文档或 Vivado 属性获取真实 `CONFIG.*` 参数；
- 生成可复现的 Tcl 配置；
- 检查参数之间的依赖关系；
- 验证端口方向、位宽、时钟和复位；
- IP 配置失败时优先读取 Vivado 错误反馈；
- 不凭记忆猜测 IP 属性名；
- 不在未确认的情况下升级 IP 版本；
- 修改后执行端口、参数和 elaboration 检查。

## 10. RTL 检查和综合问题

RTL lint 使用 `vivado-rtl-lint`，以 RTL lint 生成的报告为主要依据，并区分语法问题、综合风险、CDC 风险和功能风险。lint 无报错不代表功能正确。

Elaboration 或综合错误使用 `vivado-rtl-elaboration-analysis`，读取实际报错文件和行号，检查对应 RTL、参数、端口和实例化上下文，并给出与实际代码匹配的修改建议。

## 11. 时序和方法学检查

涉及 `report_methodology`、UG906、TIMING-1 至 TIMING-57、时序违规、时钟、CDC、约束或方法学警告时，使用 `vivado-timing-methodology-checks`。

- 先读取对应检查项的参考文档；
- 先确认违规属性和适用条件；
- 按检查组和优先级处理；
- AUTO 类修复必须验证适用条件；
- USER 类修复必须说明设计意图并取得确认；
- 不猜测时钟关系、例外路径或约束意图；
- 每组修复后重新运行 `report_methodology`；
- 约束修改后生成差异记录；
- 推荐重新综合或实现验证；
- 未完成验证的违规保持 open。

## 12. 仿真分层策略

按以下顺序推进：

1. 单模块：状态机、字段映射、计数和握手；
2. 模块链路：FIFO、适配层、PHY 数据接口和复位；
3. GT/PHY 集成：GT 支持模型、串行回环和远端模型；
4. 系统场景：真实业务长度、连续包、错误包和异常复位。

当 TX 和 RX 的线格式不同时，不直接把 TX 原始输出当作 RX 输入。应建立远端转换模型，或分别验证 TX 输出和 RX 注入输入，并在验证记录中说明覆盖边界。

仿真完成后记录编译命令、elaboration 命令、仿真命令、使用的 fileset、库和模型、通过的断言和内容比对、未覆盖场景、波形和日志位置，以及 `PASS`、`PARTIAL` 或 `BLOCKED` 结论。

## 13. 仿真和验证证据

功能结论必须至少有一种明确证据：

- SystemVerilog assertion；
- 输入输出内容比对；
- 报文计数；
- 状态机状态覆盖；
- 边界长度测试；
- 错误包测试；
- 复位恢复测试；
- FIFO 满、空、almost full、almost empty 测试；
- 波形中的握手和数据对齐；
- Vivado 报告中的实际结果。

“编译成功”“没有明显报错”“波形看起来正常”或“理论上应该正确”不能单独作为功能结论。

## 14. Vivado 工程版本控制

涉及工程可移植、Git 管理、CI/CD、自动重建、团队交接、导出 RTL/IP/BD/约束/仿真文件或生成可复现构建脚本时，使用 `vivado-revision-control`。

标准流程：

1. 检测工程类型；
2. 分析源文件位置；
3. 确认是否需要保留特定 IP/BD 的生成输出；
4. 导出 RTL、约束、IP Tcl、BD Tcl、仿真和数据文件；
5. 捕获工程设置；
6. 生成 `build.tcl`；
7. 在干净目录验证重建；
8. 记录顶层、Block Design、IP 数量和 run 是否一致。

对于 DFX、BD Container 或 Versal Segmented Configuration，先读取对应参考文档，再执行导出和重建。

代理驱动 Vivado 时，不调用需要交互式 stdin 的保留输出提示过程。应先查询可保留的 IP/BD，再向用户询问选择和外部缓存目录。

## 15. 任务清单和 Checklist

每个多阶段任务建立或更新以下记录：

```markdown
## Todo

- [open] 阅读并确认协议、帧格式和字段映射。
- [open] 确认时钟、复位、FIFO、CDC 和接口握手。
- [open] 完成 RTL、调用点和公共接口分析。
- [open] 修改 RTL、IP/TCL 或仿真文件。
- [open] 完成编码、端口、lint 和 elaboration 检查。
- [open] 建立并执行阶段性验证。
- [open] 验证边界、复位、错误和握手场景。
- [open] 保存日志、波形和报告。
- [open] 更新项目进度记录。

## Checklist

- [ ] 约束已由用户确认并写入项目文档。
- [ ] RTL 接口和文档一致。
- [ ] 所有输入输出方向已核对。
- [ ] FIFO/CDC 使用有效信号判断数据可用。
- [ ] 状态机计数边界已验证。
- [ ] 仿真 fileset、IP 网表和库完整。
- [ ] lint、elaboration 或综合检查已完成。
- [ ] 仿真结果具有断言、计数或内容比对证据。
- [ ] ILA/VIO 报告包含实际采样值。
- [ ] 工程脚本可以复现构建。
- [ ] 日志、波形和报告已保存。

## 关闭规则

- [closed] 只有有证据支持的任务才能关闭。
- [open] 失败、未覆盖、缺少模型或等待外部资料的任务保持 open。
- [blocked] 只有阻塞原因已明确且无法通过当前授权范围解决时使用。
```

## 16. 进度记录

对于多阶段、可暂停或需要交接的任务，在项目根目录维护：

`docs_<agent-name>/progress.md`

Codex 默认使用：

`docs_codex/progress.md`

每次阶段完成、暂停或交接时追加一条简短事实记录，包含：

- 时间和时区；
- 当前状态；
- 是否可以继续；
- 已完成任务；
- 验证结果；
- 未解决问题；
- 文件和报告位置；
- 下一步动作。

不要粘贴完整日志、密码、凭据或大段临时输出。

## 17. 结果报告格式

```markdown
## Verification

- 使用的 MCP/Skill：
- Vivado 版本：
- 目标器件：
- 工程或会话：
- 编译命令：
- elaboration 命令：
- 综合/实现命令：
- 仿真命令：
- 通过项：
- 未覆盖项：
- 阻塞项：
- 日志位置：
- 波形位置：
- 报告位置：
- 结论：PASS / PARTIAL / BLOCKED
```

## 18. 工具失败处理

如果 MCP、Vivado、仿真器或脚本调用失败：

1. 保存实际错误信息；
2. 检查会话、版本、工程状态和文件路径；
3. 使用对应 Skill 的参考文档查询命令或属性；
4. 进行安全的只读诊断；
5. 不用不同猜测语法反复重试；
6. 如果仍然无法解决，报告实际阻塞原因、已完成检查和下一步需要的外部条件。

任何工具都不得被描述为已经成功执行，除非返回结果或报告提供了证据。

## 19. 最终行为准则

- 以当前项目文件和实际工具输出为依据；
- 优先使用可用的 AMD Vivado MCP 和对应 Skill；
- 保持公共接口和用户已有修改；
- 在修改前明确假设和影响范围；
- 在仿真或硬件操作前说明命令、范围和预期结果；
- 用报告、断言、计数、内容比对或波形证明功能结论；
- 将未验证、未覆盖和被阻塞的内容保持为 open 或 blocked；
- 不伪造工具调用、验证结果或完成状态。
