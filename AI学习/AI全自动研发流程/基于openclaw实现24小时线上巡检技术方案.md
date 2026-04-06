> **基于 OpenClaw 当前已有的 Gateway、Heartbeat、Cron、Webhook、Sub-Agent 能力，搭建一个多-Agent 协同的 7×24 线上监控巡检系统**。我会按“架构、Agent 分工、运行机制、数据流、配置思路、容灾与安全、实施步骤”来写，并把 **OpenClaw 当前能力边界和已知限制** 一并说明。OpenClaw 目前明确支持多 Agent 路由、每个 Agent 独立 workspace / agentDir / session，支持 Gateway 调度、Heartbeat、Cron、Webhook 和 Sub-Agent 隔离运行，这些正好可以作为方案底座。

# 一、方案目标

这个方案不是做一个“会聊天的机器人”，而是做一个**持续运行的巡检控制面**：它 24 小时接收监控、日志、告警和业务信号，先做低成本规则判断，再由不同领域 Agent 并行分析，最后输出 **告警结论、归因说明、处置建议、任务派发**，形成可追踪的巡检闭环。Heartbeat 适合做常规批量巡检，Cron 适合做精确时间任务和固定班次巡检；两者结合，是 OpenClaw 官方文档建议的高效自动化模式。

------

# 二、基于 OpenClaw 现状的总体架构

建议采用 **“1 个 Gateway + 1 个 Orchestrator 主控 Agent + N 个领域专家 Agent + 外部监控/告警系统”** 的架构。OpenClaw 的 Gateway 本身就是控制平面，负责 channel、session、cron、webhook 和路由；多 Agent 模式下，每个 Agent 都有独立的 workspace、state 目录和 session 存储，天然适合把“平台巡检、应用巡检、日志诊断、业务巡检、报告归档”拆给不同 Agent。

推荐把体系拆成 4 层：

**1）信号接入层**
 Prometheus / Alertmanager、Grafana、ELK / Loki、APM、K8s 事件、业务指标平台、工单系统、IM 通知。外部系统通过 Webhook 打进 OpenClaw，或者由 OpenClaw Heartbeat / Cron 主动拉取。Webhook 是官方支持的入口，且文档明确建议把 hook 端点放在 loopback、tailnet 或可信反向代理后面。

**2）调度编排层**
 由 Gateway 的 **Heartbeat + Cron + Webhook** 组成。Heartbeat 负责周期性巡检批处理，Cron 负责精确时间点任务、日报、班次巡检和补偿任务，Webhook 负责外部系统实时触发。Cron 的调度状态和运行历史可通过 `openclaw cron status / list / runs` 观测。

**3）智能分析层**
 由一个主控 Agent 调用 `sessions_spawn` 或 `/subagents spawn` 派生多个隔离子任务。OpenClaw 当前已经支持 Sub-Agent 非阻塞运行、完成后回传总结；子 Agent 彼此默认隔离，适合做并发诊断。

**4）执行闭环层**
 将分析结果写入 IM、工单、报告仓库、值班群、知识库，必要时回推到事件系统。OpenClaw 自身可以通过 channel 回复，也可以用 webhook / 外部脚本继续分发到 Jira、飞书、Slack 或内部平台。

------

# 三、核心设计原则

第一条原则是 **“规则前置，LLM 后置”**。因为巡检是 24 小时运行的，如果每次都直接唤起大模型，成本和噪音都会很高。OpenClaw 目前已经有 Cron `gate` 机制的设计与实践案例：先跑一个确定性的 shell/tool 检查，通过才进入 LLM 分析，否则直接记为 skipped，这非常适合巡检场景。

第二条原则是 **“批量巡检走 Heartbeat，重诊断走 Sub-Agent，精确定时走 Cron”**。官方文档明确建议 Heartbeat 用来做 routine monitoring，Cron 用来做 precise schedules，二者结合最省成本。

第三条原则是 **“主控只负责编排，不负责所有分析”**。主控 Agent 只做任务切分、优先级排序、结果聚合和最终输出；真正的 CPU/Token 大头由领域 Agent 并发消化。OpenClaw 的 sub-agent 模式就是为“research / long task / slow tool 并行化”设计的。

第四条原则是 **“高风险环境按 trust boundary 拆部署”**。OpenClaw 官方安全文档明确说它不是 hostile multi-tenant 安全边界；如果存在互不信任的租户或团队，应该拆成独立 gateway / OS user / host。你这个“线上巡检”通常涉及生产凭证和运维工具，更要遵守这个边界。

------

# 四、Agent 角色设计

建议至少拆成 6 类 Agent。

## 1. Orchestrator 主控 Agent

负责汇总 Heartbeat、Cron、Webhook 触发的任务；按告警等级、来源、环境、时间窗做去重与分派；调用 `sessions_spawn` 并发拉起子 Agent；最后汇总子结果并输出统一结论。OpenClaw 的 `sessions_spawn` 支持指定 model、thinking、timeout、session/thread 模式，适合主控精细调度。

## 2. Infra 巡检 Agent

负责主机、容器、K8s、网络、磁盘、CPU、内存、Pod 重启、节点异常、部署变更等基础设施问题。它主要处理 Prometheus、K8s event、node exporter、容器日志等信号。

## 3. App 诊断 Agent

负责接口错误率、慢请求、超时、依赖失败、线程池/连接池问题、APM trace 聚合、变更前后对比等应用层问题。它更适合读取 APM 与服务日志。

## 4. Log / RCA Agent

专门做日志聚类、关键异常提取、时间线重建、根因假设生成，并给出需要进一步验证的证据点。这个 Agent 适合在异常发生后被按需拉起，而不是持续常驻。

## 5. Biz 健康 Agent

负责业务侧巡检，比如支付成功率、下单转化、消息到达率、GMV、活跃、漏斗、地区/版本维度异常。因为线上事故不一定表现为系统挂掉，也可能是业务指标静默异常。

## 6. Report / Knowledge Agent

负责日报、交接班、事故复盘初稿、SOP 引用、历史事件相似性比对和知识库归档。

这种设计和 OpenClaw 的多 Agent 隔离模型是一致的：每个 Agent 拥有自己独立的 workspace、agentDir 和 session，便于维护不同的 prompt、工具清单和知识上下文。

------

# 五、7×24 巡检运行机制

## 模式 A：Heartbeat 做基础批巡检

给主控 Agent 配置 Heartbeat，例如每 10 分钟或 15 分钟运行一次。Heartbeat 默认就是 OpenClaw 的主动模式，官方文档说明默认每 30 分钟会读取 `HEARTBEAT.md` 并执行检查。你可以把 HEARTBEAT.md 写成“轻量巡检清单”，只做批量、低成本、低风险检查。

例如 HEARTBEAT.md 可定义：

- 检查近 15 分钟是否有新的 P1/P2 告警
- 检查是否存在持续 5 分钟以上的错误率抬升
- 检查是否有 pending 工单未跟进
- 检查当天变更服务是否出现新异常
- 若无异常，输出 `HEARTBEAT_OK`
- 若发现异常，只做分类和任务分派，不直接展开长分析

这样主控 Agent 在每次 heartbeat 中，不是自己分析全部问题，而是做一次**“告警收敛器”**。

## 模式 B：Cron 做精确巡检与报告

Cron 更适合这些任务：

- 每天 09:00 生成晨检报告
- 每晚 23:00 生成值班交接摘要
- 每 30 分钟做一次数据库慢 SQL 汇总
- 每周一 10:00 做稳定性周报
- 某个事故发生后 2 小时自动提醒补复盘

官方文档说明 Cron 运行在 Gateway 内部，任务会持久化到 `~/.openclaw/cron/`，重启不会丢；支持 main session 或 isolated session 两种执行风格。对巡检来说，建议大部分定时任务都用 isolated session，避免污染主会话上下文。

## 模式 C：Webhook 做实时事件触发

当 Alertmanager、Grafana OnCall、Jenkins、ArgoCD、APM 平台、风控系统有实时事件时，直接通过 webhook 打到 OpenClaw。Webhook 支持把外部消息作为 agent payload 输入，也支持为单次触发覆盖模型。这样就能做到“有异常立刻触发分析，无异常只做低频巡检”。

------

# 六、一次异常巡检的标准处理链路

这里给你一条最实用的链路。

**第 1 步：外部事件进入 Gateway**
 比如 Alertmanager 发来“支付服务 5xx 错误率 8% 持续 5 分钟”的 webhook。Gateway 接收后，把请求送给主控 Agent。为了安全，hook 建议只开放在内网或可信代理后面，并配置独立 token。

**第 2 步：主控 Agent 做轻量归一化**
 主控把事件标准化成内部事件模型，例如：

```
{
  "env": "prod",
  "service": "payment-api",
  "severity": "P1",
  "signal_type": "apm_alert",
  "window": "5m",
  "symptom": "5xx_error_rate_high",
  "related_change": true
}
```

随后做去重、压缩、相似事件合并、事故升级判断。

**第 3 步：主控并发拉起领域 Agent**
 主控使用 `sessions_spawn` 同时启动：

- Infra Agent：检查节点/容器/网络/依赖
- App Agent：检查接口、trace、错误分布、部署版本
- Log Agent：抽取近 15 分钟异常栈、聚类、Top pattern
- Biz Agent：判断是否已影响支付成功率、订单量

OpenClaw 的 sub-agent 是非阻塞的，完成后会把摘要结果回传到请求方 session，非常适合这种并发巡检。

**第 4 步：主控聚合与交叉验证**
 主控等待子 Agent 回报，然后做证据交叉验证。例如：

- Infra 说节点正常
- App 说错误集中在 `/pay/confirm`
- Log 说最近部署后出现 `NullPointerException`
- Biz 说支付成功率已从 92% 掉到 67%

主控最终生成结论：“更可能是应用层新版本引入空指针，而不是基础设施故障。”

**第 5 步：生成处置动作**
 输出分三类：

- 值班群告警摘要
- Jira / 飞书任务
- 复盘草稿 / 观察清单

如果你的团队对自动执行比较保守，这里先停在“建议+任务”；如果自动化程度更高，可以再接回滚、扩容、重启等自动执行系统，但这一步要单独做审批闸门。

------

# 七、建议的数据与工具接入方式

我建议把接入分成三类。

## 1. Pull 型数据源

由 Heartbeat / Cron 主动拉取：

- Prometheus API
- Grafana Dashboard 快照
- Elasticsearch / Loki 查询
- K8s API
- CMDB / 发布记录
- 内部业务指标 API

这类数据适合“定时拉取 + 规则前置 + 异常再分析”。

## 2. Push 型事件源

由外部系统 webhook 推送：

- Alertmanager
- CI/CD 发布事件
- APM 告警
- 工单状态变化
- 客服高优反馈

这类更适合实时触发。

## 3. 知识与上下文源

每个 Agent 的 workspace 维护各自的：

- SOP
- 巡检 checklist
- 常见根因库
- 服务依赖拓扑
- 排障手册
- 历史事故摘要

OpenClaw 的 workspace 是每个 Agent 的默认上下文和文件工作目录，适合这样组织。

------

# 八、配置与目录组织建议

建议目录按 Agent 拆：

```
~/.openclaw/
├── openclaw.json
├── workspace-orchestrator/
│   ├── AGENTS.md
│   ├── HEARTBEAT.md
│   ├── SOP/
│   └── prompts/
├── workspace-infra/
├── workspace-app/
├── workspace-log/
├── workspace-biz/
└── agents/
    ├── orchestrator/
    │   ├── agent/
    │   └── sessions/
    ├── infra/
    ├── app/
    ├── log/
    └── biz/
```

这样每个 Agent 的 prompt、技能、知识、认证和 session 都是隔离的，符合 OpenClaw 的推荐模型：不要复用 agentDir，否则会导致 auth / session 冲突。

------

# 九、推荐的多-Agent 分工与模型策略

主控 Agent 用高质量模型，子 Agent 用低成本模型。OpenClaw 文档明确建议：sub-agent 会有独立上下文和 token 消耗，重或重复任务应该使用更便宜的模型。

一个实用分配是：

- 主控 Agent：高质量模型，负责编排和最终总结
- Infra / Log Agent：中低成本模型，强调结构化输出
- Biz Agent：中成本模型，强调业务解释能力
- Report Agent：低成本模型，偏整理归档

同时给 sub-agent 设 `runTimeoutSeconds`，例如 60~120 秒，防止单个分析长时间卡住。OpenClaw 的 `sessions_spawn` 支持单次 timeout。

------

# 十、巡检任务分类设计

建议把任务分成 4 级。

## L0：纯规则检查

不进 LLM，直接用脚本或 API 检查，例如：

- CPU > 90% 持续 10 分钟
- 某指标环比跌幅 > 20%
- Pod restart > 3
- 消息堆积 > 阈值

这种适合放在 Cron gate 或外部预处理层。文档和社区实践都说明 gate 是为了避免无意义的 LLM 调用。

## L1：轻量事件归类

进入主控 Agent，但不拉子 Agent，只输出告警归类和建议动作。

## L2：标准并发诊断

主控 + 2~4 个子 Agent 并行分析，产出 RCA 候选和处置建议。

## L3：升级事故模式

除并发诊断外，再触发：

- 分时汇报
- 事故房间总结
- 复盘模板
- 值班升级通知

------

# 十一、观测性与审计设计

要让这个系统真的能值班，必须把它自己也纳入观测。

至少监控这些指标：

- Heartbeat 最近一次是否成功
- Cron 任务数、下一次唤醒时间、最近 20 次运行状态
- Webhook 请求量、失败率、拒绝次数
- Sub-agent 运行数、成功率、超时率
- 模型调用 token、成本、平均时延
- 每类巡检结论的误报率、漏报率、确认率

OpenClaw 提供了 `openclaw cron status`、`openclaw cron runs`、`openclaw system heartbeat last`、`openclaw logs --follow` 这类排查入口，可以纳入自监控 SOP。

------

# 十二、当前 OpenClaw 的关键限制与规避方案

这里有一个你必须注意的点：**OpenClaw 当前多-Agent 场景下，Cron 的 agent 级隔离并不完善**。GitHub 上已有相关 issue 指出，cron job 目前是 gateway 级而不是 agent 级，在多 Agent / 多用户场景里可能出现跨 agent 可见、可编辑、甚至在别的 agent 上下文执行的风险。相关 issue 到 2026 年 3 月仍可见，其中一个是 closed as stale，另一个仍是 open enhancement。

所以，你这个“24 小时巡检”方案里，**不要把所有 Agent 都开放 cron 权限**。我建议：

- 只允许 **主控 Agent / 平台运维 Agent** 管理 Cron
- 其他领域 Agent 只允许被动由 `sessions_spawn` 拉起
- 常规巡检尽量用 agent-scoped 的 Heartbeat
- 如果必须做到强隔离租户，按 OpenClaw 安全文档建议拆多个 gateway / host / OS user

这套规避方式更符合当前产品能力边界。

------

# 十三、安全方案

OpenClaw 官方明确强调：它默认是“个人助理/单信任边界”模型，而不是对抗式多租户平台。因此生产巡检部署要额外收口权限。

建议你至少做这几件事：

1. **一个生产环境一个 gateway 或至少一个 OS user**，不要把多个互不信任团队塞进同一个高权限 gateway。
2. **Webhook 只走内网或可信代理**，使用独立 token，不复用 gateway token。
3. **高危执行工具默认禁用**，比如 shell、浏览器控制、写操作工具，只给特定 Agent 开。
4. **workspace 只放必要知识，不放全集 secrets**。
5. **运行 `openclaw security audit` 定期审计**，官方文档建议改完配置后定期执行。
6. **所有自动处置动作走审批闸门**，尤其是回滚、扩容、重启、数据库操作。

------

# 十四、最小可行版本（MVP）怎么做

如果你现在刚起步，不建议一上来做全量巡检。我建议按 3 个阶段推进。

## Phase 1：单场景 PoC

只做一个服务，例如 `payment-api`：

- Heartbeat 每 15 分钟巡检一次
- 接入 Alertmanager webhook
- 主控 Agent + App Agent + Log Agent
- 输出到飞书值班群
- 不做自动执行，只做分析和建议

这样能最短时间验证“巡检链路通不通”。

## Phase 2：多领域并发诊断

增加：

- Infra Agent
- Biz Agent
- 晨报/晚报 Cron
- 事故升级逻辑
- 基础误报抑制

## Phase 3：闭环自动化

增加：

- Jira / 工单联动
- SOP 自动引用
- 历史事故相似性检索
- 复盘草稿生成
- 审批后自动执行处置脚本

这条路线和 OpenClaw 当前能力匹配度最高，也最容易先出结果。

------

# 十五、给你一个推荐的实施版本

如果你要我直接给一个“最靠谱”的落地版本，我建议是：

**Heartbeat + Webhook 作为入口，Orchestrator 作为唯一编排者，领域 Agent 全部通过 `sessions_spawn` 被动拉起，Cron 仅由主控持有，用于日报、补偿任务和精确定时巡检。** 这样能同时利用 OpenClaw 的多 Agent、Sub-Agent、Heartbeat、Cron 和 Webhook 能力，又能规避当前 cron 多 Agent 隔离不足的问题。

------

# 十六、你可以直接拿去汇报的一句话

**基于 OpenClaw 的 Gateway、Heartbeat、Cron、Webhook 与 Sub-Agent 能力，构建“主控编排 + 领域专家并发诊断”的 7×24 线上巡检体系，实现异常感知、交叉分析、告警输出与任务闭环的一体化智能巡检。**