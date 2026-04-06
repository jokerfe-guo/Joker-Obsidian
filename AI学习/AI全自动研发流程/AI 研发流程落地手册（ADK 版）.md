## 基于 Google ADK 5种 Agent Skill 设计模式的前端团队规范化迭代指南

> **适用团队**：React + NestJS 前端团队 · 约 30 人 · 无专职架构师
> **核心目标**：统一 AI 代码生成规范 · 保证架构稳定性 · 提高迭代质量 
> **参考框架**：[Google Agent Development Kit (ADK)](https://google.github.io/adk-docs/) · [5 Agent Skill Design Patterns](https://x.com/GoogleCloudTech/status/2033953579824758855) 
> **文档版本**：ADK 版（对比版本见附录：与 OpenSpec 的详细对比）

---

## 目录

1. [我们在解决什么问题](#1-%E6%88%91%E4%BB%AC%E5%9C%A8%E8%A7%A3%E5%86%B3%E4%BB%80%E4%B9%88%E9%97%AE%E9%A2%98)
    
2. [Google ADK 5种 Agent Skill 设计模式](#2-google-adk-5%E7%A7%8D-agent-skill-%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F)
    
3. [前端架构规范体系（Tool Wrapper 技能包）](#3-%E5%89%8D%E7%AB%AF%E6%9E%B6%E6%9E%84%E8%A7%84%E8%8C%83%E4%BD%93%E7%B3%BBtool-wrapper-%E6%8A%80%E8%83%BD%E5%8C%85)
    
4. [OpenSpec vs ADK：两套方案深度对比](#4-openspec-vs-adk%E4%B8%A4%E5%A5%97%E6%96%B9%E6%A1%88%E6%B7%B1%E5%BA%A6%E5%AF%B9%E6%AF%94)
    
5. [使用示例：低码项目迁移 Vibe Coding（Feature Pipeline 完整演示）](#5-%E4%BD%BF%E7%94%A8%E7%A4%BA%E4%BE%8B%E4%BD%8E%E7%A0%81%E9%A1%B9%E7%9B%AE%E8%BF%81%E7%A7%BB-vibe-codingfeature-pipeline-%E5%AE%8C%E6%95%B4%E6%BC%94%E7%A4%BA)
    
6. [日常需求迭代规范（S/M/L 三档技能组合）](#6-%E6%97%A5%E5%B8%B8%E9%9C%80%E6%B1%82%E8%BF%AD%E4%BB%A3%E8%A7%84%E8%8C%83sml-%E4%B8%89%E6%A1%A3%E6%8A%80%E8%83%BD%E7%BB%84%E5%90%88)
    
7. [团队分工与规范维护](#7-%E5%9B%A2%E9%98%9F%E5%88%86%E5%B7%A5%E4%B8%8E%E8%A7%84%E8%8C%83%E7%BB%B4%E6%8A%A4)
    
8. [成功指标与风险控制](#8-%E6%88%90%E5%8A%9F%E6%8C%87%E6%A0%87%E4%B8%8E%E9%A3%8E%E9%99%A9%E6%8E%A7%E5%88%B6)
    

---

## 1. 我们在解决什么问题

### 1.1 现状：AI 编码没有护栏的团队

当 30 人前端团队开始大量引入 AI 辅助编码，但没有统一规范时，会出现一个共同模式：每个人都在用 AI，但 AI 是完全开放的——它会按照每次聊天的上下文生成代码，而不是按照团队的架构约定来生成。

![AI编码无规范的团队痛点|697](file:///Users/guohaohao/Desktop/AI-joker/%E5%85%A8%E8%87%AA%E5%8A%A8AI%E5%BC%80%E5%8F%91%E6%B5%81%E7%A8%8B/adk-handbook/images/01_pain_points.svg?lastModify=1774875540)

**具体表现：**

- **代码风格割裂**：同一个 Chart 组件，有人用 `useEffect` 直接在组件里调 API，有人用自定义 Hook，有人复制了别人的写法但改了一半。三个月后，同类型的表格组件全项目有 7 个版本。
    
- **文件结构混乱**：新人不知道一个新组件该放在 `components/`、`pages/` 还是 `views/`，AI 也不知道，它只是把代码放在当前目录。半年后 `src/` 下有 200 个文件打平堆放。
    
- **AI 生成代码无 Review**：因为"是 AI 写的"所以默认没问题，但 AI 生成的代码里带着 `any` 类型、内联 style、直接 fetch 调用，这些都悄悄进入了生产代码。
    
- **架构规则靠口耳相传**：TL 说"图表要用封装过的 BaseChart"，但这个规则只存在于入职培训的 PPT 里，AI 当然不知道，新来的成员也可能忘记。
    

**根本原因**：AI 是一个没有"组织记忆"的工具。每次对话都是全新开始，它不知道你们团队的规范、你们项目的架构决策、你们过去踩过的坑。

### 1.2 目标：给 AI 装上"组织记忆" + "行为边界"

这套方案的核心是：**把团队规范写进 SKILL.md，让 AI 按需加载，按模式执行**。

|目标|解决方式|
|---|---|
|AI 代码生成规范统一|Tool Wrapper：把框架/组件库规范装进技能包，AI 用什么加载什么|
|需求不被误解乱执行|Inversion：AI 先采访完需求，收集完整信息才开始写代码|
|文档格式统一、人人能写|Generator：模板驱动，填空式生成 spec/design/task 文档|
|代码质量有量化标准|Reviewer：按 CRITICAL→LOW 分级评分，TL 只看关键级别|
|大需求不失控|Pipeline：多阶段检查点，防止 AI 跳步骤|

---

## 2. Google ADK 5种 Agent Skill 设计模式

### 2.1 什么是 Agent Skill

**Agent Skill** 是 Google Agent Development Kit（ADK）提出的 AI 能力封装单元。其核心思想是：

> 把 AI 的知识、规则、行为约束写进结构化的 `SKILL.md` 文件及其配套资源，让 Agent 在需要时**按需加载**，而不是把所有规则塞进 system prompt。

一个 Skill 的基本目录结构：

my-skill/  
├── SKILL.md           # 核心指令：AI 在执行此技能时的行为约束（必填）  
├── assets/            # 输出模板、数据文件（可选）  
│   └── template.md  
└── references/        # 参考资料：API 文档、Checklist、风格指南（可选）  
    └── guide.md

`SKILL.md` 的典型结构：

---  
name: react-expert  
description: 当工作涉及 React 组件开发时激活  
triggers:  
  - "创建组件"  
  - "写 React"  
  - "hooks"  
---  
​  
# React 开发规范（技能激活后 AI 必须遵守）  
​  
## 你的角色  
你是遵守本团队 React 规范的前端开发者。  
​  
## 强制规则  
- 所有图表必须通过 src/components/charts/ 封装层，禁止直接使用 ECharts  
- 禁止 inline style，所有样式使用 Tailwind CSS 类名  
- 异步操作必须有完整的 loading / error / empty 三态处理  
...

### 2.2 5种设计模式总览

Google 在大量 ADK 项目中提炼出 5 种反复出现的技能设计模式，覆盖了 AI 辅助开发的全生命周期：

---

### 2.3 模式一：Tool Wrapper（工具包装器）

**核心思想**：把特定库或框架的知识封装成技能包，让 Agent 在实际用到这个技术时才加载，不污染全局上下文。

**适用场景**：技术栈多、规范细的团队。你们同时有 React、NestJS、ECharts，每个都有各自的使用约定。

**工作原理**：

1. TL 把每个库/框架的使用规范写进对应的 `SKILL.md` 和 `references/`
    
2. 开发者在 prompt 里描述任务，AI 识别到"这是 React 任务"
    
3. AI **按需**加载 `react-expert` skill，读取规范后再动手
    
4. 不涉及 NestJS 的任务，`nestjs-expert` 不会被加载，不占用上下文
    

**对比传统方式**：传统方式把所有规范堆在 system prompt 里，上下文很快被规则占满，AI 反而更容易忽略其中某条。

---

### 2.4 模式二：Generator（文档生成器）

**核心思想**：从可复用的模板出发，填空式生成结构化文档。分离"内容收集"和"格式定义"，保证 每个人都产出格式一致的文档。

![Generator 模式|697](file:///Users/guohaohao/Desktop/AI-joker/%E5%85%A8%E8%87%AA%E5%8A%A8AI%E5%BC%80%E5%8F%91%E6%B5%81%E7%A8%8B/adk-handbook/images/04_generator.svg?lastModify=1774875540)

**适用场景**：需求 spec 文档、技术设计文档、任务清单——任何需要标准化格式的文档产物。

**工作原理**：

1. `assets/spec-template.md` 定义文档框架（有哪些章节、每节写什么）
    
2. `references/style-guide.md` 定义写作规范（用语标准、格式约定）
    
3. AI 拿到模板，逐项向开发者提问收集信息
    
4. 填充完成，输出格式统一的文档
    

**两个配套 Generator**：

generators/  
├── spec-generator/      # 需求规格文档（含功能说明、影响范围、验收标准）  
├── design-generator/    # 技术设计文档（含架构选型、关键实现、风险评估）  
└── task-generator/      # 任务清单（含子任务分解、工时估算、依赖关系）

---

### 2.5 模式三：Reviewer（代码审查员）

**核心思想**：把"检查什么"（checklist）和"如何检查"（SKILL.md 指令）分开，让 AI 按严重级别逐项评分，输出结构化报告。

![Reviewer 模式|697](file:///Users/guohaohao/Desktop/AI-joker/%E5%85%A8%E8%87%AA%E5%8A%A8AI%E5%BC%80%E5%8F%91%E6%B5%81%E7%A8%8B/adk-handbook/images/05_reviewer.svg?lastModify=1774875540)

**适用场景**：PR 合并前的代码质量把关，替代人工逐行 Review 的重复劳动。

**Severity 分级**：

|级别|含义|处理方式|
|---|---|---|
|**CRITICAL**|阻塞性问题（安全漏洞、架构违规）|必须修复才可合并|
|**HIGH**|重要问题（类型错误、性能隐患）|强烈建议在合并前修复|
|**MEDIUM**|规范问题（命名不规范、缺少注释）|建议修复|
|**LOW**|优化建议（冗余代码、可读性改进）|下次迭代处理|

**TL 工作流**：只需关注 CRITICAL 和 HIGH 级别，大幅降低 Review 人力成本。Reviewer 的另一个优势是**可热换 checklist**——把 `react-checklist.md` 换成 `security-checklist.md`，立即变成安全审计工具。

---

### 2.6 模式四：Inversion（反转采访）

**核心思想**：反转传统的"用户提 prompt → AI 执行"模式，让 AI 先扮演采访者，收集完所有信息才开始动手。SKILL.md 里包含明确的**不可逾越的门控指令**（"DO NOT generate any code until all phases are complete"）。

![Inversion 模式|697](file:///Users/guohaohao/Desktop/AI-joker/%E5%85%A8%E8%87%AA%E5%8A%A8AI%E5%BC%80%E5%8F%91%E6%B5%81%E7%A8%8B/adk-handbook/images/06_inversion.svg?lastModify=1774875540)

**为什么需要这个模式**：AI 最大的问题之一是太急于执行。开发者说"帮我做个图表筛选功能"，AI 立刻开始写代码，但可能误解了"筛选"的含义，等写完才发现方向不对，重写成本极高。

**三阶段采访协议**：

# Inversion SKILL.md 核心指令（示例）  
​  
**门控规则（AI 必须严格遵守）**：  
在以下所有阶段完成前，你不允许生成任何代码或技术方案。  
​  
## Phase 1：业务上下文（必须全部回答）  
- 这个需求解决什么用户问题？  
- 谁是使用者，使用场景是什么？  
- 有没有已知的边界情况或限制？  
​  
## Phase 2：技术范围（必须全部回答）  
- 涉及哪些页面/组件/接口？  
- 数据从哪里来，格式是什么？  
- 有无前置依赖项或现有逻辑需要复用？  
​  
## Phase 3：约束与验收（必须全部回答）  
- 性能要求？（是否有加载时间限制）  
- 截止时间？（影响技术选型）  
- 如何验收？（成功的标准是什么）  
​  
✅ 全部回答完毕后，调用 spec-generator 生成需求文档。

---

### 2.7 模式五：Pipeline（工作流管道）

**核心思想**：把多个模式串联成有检查点的严格工作流，每个阶段有明确的进入条件和完成条件，AI 不能跳过任何步骤。

![Pipeline 模式|641](file:///Users/guohaohao/Desktop/AI-joker/%E5%85%A8%E8%87%AA%E5%8A%A8AI%E5%BC%80%E5%8F%91%E6%B5%81%E7%A8%8B/adk-handbook/images/07_pipeline.svg?lastModify=1774875540)

**完整 Feature Pipeline**（M/L 档需求适用）：

Stage 1: Inversion（需求采访）  
    输入：需求意图  
    输出：完整的 context 信息  
    完成条件：3 个阶段的采访问题全部回答  
         ↓ Checkpoint 1: spec.md 完整存在？  
Stage 2: Generator（生成设计文档）  
    输入：spec.md  
    输出：design.md + task-list.md  
    完成条件：TL 确认设计方案  
         ↓ Checkpoint 2: design.md 技术方案已 TL 确认？  
Stage 3: Tool Wrapper（加载技术规范）  
    输入：design.md 确定的技术栈  
    输出：激活对应的 Wrapper Skill  
    完成条件：相关技术规范已加载  
         ↓ Checkpoint 3: 规范已就绪，可以开始实施？  
Stage 4: 代码实施  
    输入：task-list.md + 激活的 Wrapper Skill  
    输出：实现代码  
    完成条件：所有 task checkbox 勾选完毕  
         ↓ Checkpoint 4: 所有任务完成？  
Stage 5: Reviewer（质量审查）  
    输入：实现代码  
    输出：CRITICAL/HIGH/MED/LOW 分级报告  
    完成条件：CRITICAL = 0（否则回到 Stage 4 修复）

**Bugfix Pipeline**（简化版，Bug 修复适用）：

Stage 1: 问题复现描述（Inversion mini 版，1 个阶段）  
    ↓ Checkpoint: 复现步骤清晰？  
Stage 2: 根因分析 + 修复实施（Tool Wrapper 激活）  
    ↓ Checkpoint: 修复已验证？  
Stage 3: Reviewer（回归检查）  
    完成条件：CRITICAL = 0 且无高风险副作用

---

## 3. 前端架构规范体系（Tool Wrapper 技能包）

### 3.1 技能包目录总览

这是替代原来"口耳相传规范"的核心资产，全部存放在项目根目录的 `.skills/` 下，随代码一起进 Git。

![.skills 目录结构|697](file:///Users/guohaohao/Desktop/AI-joker/%E5%85%A8%E8%87%AA%E5%8A%A8AI%E5%BC%80%E5%8F%91%E6%B5%81%E7%A8%8B/adk-handbook/images/08_skills_directory.svg?lastModify=1774875540)

```
.skills/  
├── tool-wrappers/          # 技术栈专家包  
│   ├── react-expert/       # React 组件开发规范  
│   ├── nestjs-expert/      # NestJS BFF 开发规范  
│   └── chart-expert/       # 图表封装层规范（重点）  
├── generators/             # 文档生成器  
│   ├── spec-generator/     # 需求规格文档  
│   ├── design-generator/   # 技术设计文档  
│   └── task-generator/     # 任务清单  
├── reviewers/              # 代码审查员  
│   ├── react-reviewer/     # React 代码评审  
│   └── nestjs-reviewer/    # NestJS 代码评审  
├── inversions/             # 需求采访者  
│   └── requirement-intake/ # 通用需求采访流程  
└── pipelines/              # 工作流管道  
    ├── feature-pipeline/   # 完整功能开发流程  
    └── bugfix-pipeline/    # Bug 修复流程

```
---

### 3.2 三类项目的 Tool Wrapper 规格

#### 类型一：纯 React 项目（`react-expert` skill）

`SKILL.md` 核心内容：

---  
name: react-expert  
description: 纯 React 18 + TypeScript 项目开发规范  
---  
​  
## 目录结构（强制）  
src/  
├── components/        # 纯 UI 组件（无业务逻辑）  
│   ├── common/        # 通用基础组件（Button、Input 等）  
│   ├── charts/        # 图表封装层（必须从这里引用）  
│   └── layout/        # 布局组件  
├── pages/             # 页面级组件（路由对应）  
├── hooks/             # 自定义 Hook（useXxx 命名）  
├── services/          # API 调用层（统一封装）  
├── stores/            # 状态管理  
└── types/             # TypeScript 类型定义  
​  
## 组件规范（强制）  
- 所有组件必须有完整 TypeScript 类型，禁止 any  
- Props 接口命名：ComponentNameProps  
- 禁止在组件内直接调用 API（统一走 services/ + React Query）  
- 图表组件禁止直接引入 ECharts，必须通过 src/components/charts/ 封装层  
​  
## 样式规范（强制）  
- 只使用 Tailwind CSS 工具类，禁止 inline style  
- 禁止硬编码颜色值，使用 Tailwind 主题变量  
- 复杂样式抽取到独立的 CSS Module  
​  
## 图表三态（强制）  
所有数据展示组件必须实现 loading / error / empty 三态：  
- loading：Skeleton 骨架屏（使用内部组件库 Skeleton）  
- error：ErrorBoundary 包裹 + 友好提示  
- empty：EmptyState 组件（禁止直接显示空白）

`references/` 包含：

- `component-patterns.md`：常见组件模式示例（Table、Form、Chart 的标准写法）
    
- `hooks-guide.md`：自定义 Hook 命名和使用规范
    
- `internal-library.md`：内部组件库 API 速查（哪些组件存在，怎么用）
    

---

#### 类型二：NestJS BFF 项目（`nestjs-expert` skill）

`SKILL.md` 核心内容：

---  
name: nestjs-expert  
description: NestJS BFF（Backend For Frontend）层开发规范  
---  
  
## 模块结构（强制）  
```
src/  
├── modules/  
│   └── [feature]/  
│       ├── [feature].module.ts  
│       ├── [feature].controller.ts   # 路由入口，薄层，只做参数校验  
│       ├── [feature].service.ts      # 业务逻辑  
│       ├── [feature].dto.ts          # 请求/响应 DTO（class-validator 装饰器）  
│       └── [feature].types.ts        # 类型定义  
├── common/  
│   ├── interceptors/                 # 响应格式统一拦截器  
│   ├── filters/                      # 全局异常过滤器  
│   ├── guards/                       # 认证/权限守卫  
│   └── pipes/                        # 数据转换管道  
└── config/                           # 配置模块  
```
  
## DTO 规范（强制）  
- 请求 DTO 必须用 class-validator 装饰器做校验  
- 响应 DTO 必须明确定义，禁止直接透传上游接口的原始结构  
- 禁止在 Controller 直接操作数据库或调用下游服务  
  
## 错误处理规范（强制）  
- 所有 Service 方法必须有明确的错误类型  
- 业务错误使用自定义 HttpException 子类  
- 禁止在 Service 里 console.log，使用 Logger 服务

---

#### 类型三：NestJS SSR 项目（`nestjs-ssr-expert` skill）

`SKILL.md` 核心内容：

---  
name: nestjs-ssr-expert  
description: NestJS + 服务端渲染项目规范（含 SEO 要求）  
---  
  
## SSR 特殊要求（强制）  
- 所有页面组件必须实现 getServerSideProps 或 getStaticProps（视更新频率决定）  
- 禁止在 SSR 阶段访问 window / document 对象（需做 typeof window 判断）  
- API 调用需区分：SSR 阶段走服务端直连，客户端阶段走 /api 代理层  
  
## SEO 规范（强制）  
- 每个页面必须有唯一的 title / description meta  
- 图片必须有 alt 属性  
- 页面首屏内容不可依赖 client-side only 的数据  
  
## 水合（Hydration）注意事项（强制）  
- 避免服务端/客户端渲染结果不一致（Hydration mismatch）  
- 动态内容（时间戳、随机值等）必须仅在客户端渲染，用 useEffect 包裹

---

### 3.3 图表封装层（chart-expert skill 重点说明）

图表在你们项目中占比高，是最容易出现规范违规的地方。`chart-expert` skill 的 SKILL.md 要明确规定：

# 图表开发规范（AI 必须严格遵守）  
  
## 封装层结构  
```
src/components/charts/  
├── BaseChart/         # 基础图表容器（响应式、loading态、错误态）  
├── LineChart/         # 折线图（封装了团队常用配置）  
├── BarChart/          # 柱状图  
├── PieChart/          # 饼图  
├── ScatterChart/      # 散点图  
└── index.ts           # 统一导出  
```
  
## 强制规则  
- 任何业务代码中禁止 import { ECharts } from 'echarts'  
- 所有图表通过 src/components/charts/ 中的组件使用  
- 图表尺寸通过父容器控制，不允许在图表组件内硬编码 width/height px 值  
- 颜色 palette 统一从 src/theme/chartTheme.ts 读取，不允许在业务代码中写颜色值  
  
## 新增图表类型流程  
1. 先查 src/components/charts/ 是否已有同类组件  
2. 确实没有 → 在 BaseChart 基础上扩展 → 提 PR 给 TL 合并后再使用  
3. 禁止在业务组件中临时封装图表

---

## 4. OpenSpec vs ADK：两套方案深度对比

在采用 ADK 模式之前，有必要深度理解两套方案的本质差异，以便后续团队在维护和调整时有清晰的认知。

![OpenSpec vs ADK 对比|697](file:///Users/guohaohao/Desktop/AI-joker/%E5%85%A8%E8%87%AA%E5%8A%A8AI%E5%BC%80%E5%8F%91%E6%B5%81%E7%A8%8B/adk-handbook/images/10_comparison.svg?lastModify=1774875540)

### 4.1 核心理念对比

|维度|OpenSpec（规格驱动）|ADK Skill 模式（上下文驱动）|
|---|---|---|
|**哲学起点**|"先写完整规格，AI 按规格执行"|"按需加载规范，AI 按模式行动"|
|**知识组织方式**|按能力域组织 spec 文件，增量合并|按行为模式组织 skill 包，独立激活|
|**文档与规范关系**|规格文件即是文档（openspec/specs/）|技能包文件即是规范（.skills/）|
|**AI 约束方式**|通过 Delta Spec 的语义标记约束|通过 SKILL.md 的明确指令约束|

### 4.2 机制层面对比

|机制|OpenSpec|ADK Skill 模式|
|---|---|---|
|**需求进入流程**|手动写 proposal.md，提 Change|Inversion 模式：AI 主动采访，自动生成 spec|
|**文档生成**|开发者手动写 design.md、tasks.md|Generator 模式：AI 按模板填空生成|
|**增量变更追踪**|Delta Spec（ADDED/MODIFIED/REMOVED 标记）合并至主规格|无专门机制，直接更新 SKILL.md 即可|
|**代码评审**|无内置机制，依赖人工|Reviewer 模式：AI 按 checklist 分级评分|
|**命令体系**|7 个 /opsx: 命令（propose/explore/apply/archive 等）|3 个主命令（setup/newTrack/implement）等，更精简|
|**归档**|/opsx:archive 将 Delta 合并回主规格|无需归档步骤，直接维护 SKILL.md|

### 4.3 优缺点详析

#### OpenSpec 的优势

**1. 规格文件是系统的完整真相** `openspec/specs/` 里的文件在任何时候都描述系统的当前状态，新人读完即可理解系统全貌。这个"活文档"价值很高，在中大型项目中尤为突出。

**2. Delta Spec 的精确变更追踪** 每一次变更都有 ADDED/MODIFIED/REMOVED 的明确语义标记，历史可溯，在多人并行开发时能清晰看到每个 Change 改动了哪些规格。

**3. 完整的需求→设计→实施链条** openspec/changes/ 的四个 artifact（proposal + specs + design + tasks）覆盖了从"为什么做"到"怎么做"的全链路，文档结构严谨。

**4. 工具无关性强** 统一的 `/opsx:` 命令接口已对接 21+ AI 工具，理论上团队用不同 AI 工具也能执行相同流程。

#### OpenSpec 的劣势

**1. 学习曲线较陡** Delta Spec 的 ADDED/MODIFIED/REMOVED 语义对于普通前端开发者来说是一个额外的心智负担，需要专门培训才能正确使用。

**2. 缺乏代码质量闭环** OpenSpec 侧重于规格阶段，没有内置的代码质量审查机制，完成实施后的质量把关仍依赖人工。

**3. 需求采集仍是手动的** 开发者需要自己写 proposal.md，如果需求描述不完整，后面的 design 和 tasks 都会受影响，但框架本身不会主动帮你补全信息。

**4. 归档操作引入额外认知** `/opsx:archive` 的 Delta 合并步骤是很多团队容易忘记或做不规范的地方。

---

#### ADK Skill 模式的优势

**1. 模式直觉，学习成本低** Tool Wrapper / Generator / Reviewer / Inversion / Pipeline 这 5 个名字高度自解释，开发者看名字就知道这个技能做什么。对于 30 人团队来说，推广阻力更小。

**2. Inversion 解决了"AI 急于执行"的根本问题** 这是 OpenSpec 没有的能力。在 AI 写任何代码之前，强制完成需求采访，从源头减少理解偏差。

**3. Reviewer 内置代码质量闭环** AI 生成代码→AI 评审代码→输出分级报告→人工只看关键级别，形成完整的质量保障链，无需额外工具。

**4. 技能包可热换复用** 同样的 Reviewer skill，换一个 checklist 就成为安全审计、性能审计、可访问性审计，扩展性好。

**5. 无归档复杂度** 更新规范直接改 SKILL.md，不需要理解 Delta 合并机制。

#### ADK Skill 模式的劣势

**1. 缺乏"系统当前状态"的完整视图** 不像 OpenSpec 的 `openspec/specs/` 能描述整个系统，`.skills/` 里的技能包是行为规则，不是系统文档。如果需要"新人看文档理解系统"，还需要额外维护 README/Wiki。

**2. 技能包依赖 AI 正确识别和加载** Tool Wrapper 的有效性依赖于 AI 能够正确根据任务类型加载对应技能。不同 AI 工具的 skill 加载机制略有差异，需要验证。

**3. Pipeline 需要 AI 严格遵守检查点约束** Pipeline 的有效性依赖于 AI 不跳步骤，但部分 AI 工具可能会"好意地"跳过某些阶段。需要在 SKILL.md 里使用强化措辞（DO NOT、MUST NOT）。

**4. 跨 Track 的规格一致性较弱** 多个 Track 并行时，ADK 没有像 OpenSpec 那样的"主规格"来保证全局一致，需要 TL 手动维护 SKILL.md 的更新。

### 4.4 选择建议

|场景|推荐选择|原因|
|---|---|---|
|30人前端团队·快速落地|**ADK Skill 模式**|学习成本低，5个模式覆盖核心痛点，3个月内可见效果|
|需要完整系统文档追踪|OpenSpec|Delta 合并机制提供精确的规格历史|
|代码质量问题是首要痛点|**ADK Skill 模式**|Reviewer 模式直接解决，OpenSpec 无此能力|
|需求理解偏差频繁发生|**ADK Skill 模式**|Inversion 模式从源头解决|
|团队 AI 工具统一、有专职维护者|OpenSpec 也可|学习成本可以被消化|
|现有项目渐进接入|**ADK Skill 模式**|SKILL.md 可以逐步添加，无需全量改造|

**本团队最终选择**：ADK Skill 模式。主要原因：

1. 30人团队推广阻力更小（模式名自解释，3命令体系）
    
2. 代码质量和需求理解是最迫切的两个痛点，Reviewer + Inversion 直接命中
    
3. 渐进接入可行——先上 Tool Wrapper，稳定后再引入 Pipeline
    

---

## 5. 使用示例：低码项目迁移 Vibe Coding（Feature Pipeline 完整演示）

> 背景：某低代码平台项目需要迁移为 Vibe Coding（AI 驱动开发），涉及多个页面重构、数据层改造、图表系统迁移。这是一个典型的 L 档需求，完整走 feature-pipeline。

### 5.1 Stage 1：Inversion（需求采访）

**触发命令**：激活 `requirement-intake` + `feature-pipeline` skill

AI 按采访协议执行 3 阶段提问：

**Phase 1 - 业务上下文**

AI: 这次低码平台迁移的核心驱动力是什么？  
    是功能天花板（需要自定义功能低码不支持），  
    还是维护成本（低码平台升级依赖），  
    还是性能问题（渲染瓶颈）？  
  
Dev: 主要是功能天花板——低码平台不支持我们需要的复杂图表交互  
     以及组件样式无法完全定制  
  
AI: 了解。迁移范围是全量迁移还是部分页面？用户规模？  
    新版本有没有需要同时支持旧低码版的过渡期？  
  
Dev: 部分迁移，先迁 3 个高频页面...（继续对话）

**Phase 2 - 技术范围**

AI: 这 3 个页面现在的数据层是什么？是低码平台的内置数据源  
    还是已经有 API 接口？  
  
Dev: 有部分 REST API，图表数据来自 NestJS BFF 层  
  
AI: 图表用的是低码平台内置图表还是 ECharts？  
    迁移后需要保留同样的图表类型吗？  
    ...

**Phase 3 - 约束确认**

AI: 迁移过程中 3 个页面需要同时在线吗（零停机迁移）？  
    验收标准是什么——功能对齐？还是还有性能指标？  
  
Dev: 需要零停机，性能要求：首屏 LCP ≤ 2s  
    验收：3 个页面功能完全对齐现有低码版本

### 5.2 Stage 2：Generator（生成设计文档）

Checkpoint 1 通过后，AI 调用 `spec-generator` 和 `design-generator`：

**生成的 spec.md（片段）**：

# 需求规格：低码平台 3 页面迁移  
  
## 背景与目标  
- 驱动力：功能天花板（复杂图表交互 + 样式定制）  
- 目标：将 Dashboard / Analytics / Report 3 个页面迁移为 React + NestJS BFF 方案  
  
## 影响范围  
- 前端：3 个新页面组件（涉及 src/pages/ 下新增）  
- 图表：复用 src/components/charts/ 封装层，禁止直接使用 ECharts  
- BFF 层：新增 3 个 NestJS Module，改造图表数据聚合接口  
  
## 验收标准  
- [ ] 3 个页面功能与低码版本对齐（逐项对照功能清单）  
- [ ] 首屏 LCP ≤ 2s（Lighthouse 测试）  
- [ ] 零停机切换（灰度发布，用特性开关控制）

**TL 在 Checkpoint 2 审阅 design.md**，重点确认：技术选型是否合理、图表迁移方案是否符合封装层规范、BFF 模块结构是否一致。

### 5.3 Stage 3-4：工具加载 + 实施

Checkpoint 3 通过后：

- `nestjs-expert` skill 激活 → AI 了解 NestJS BFF 模块规范
    
- `chart-expert` skill 激活 → AI 知道图表只能走封装层
    
- `react-expert` skill 激活 → AI 了解组件目录结构和 TypeScript 规范
    

AI 按 `task-list.md` 逐项执行：

## 任务清单  
  
### Phase A: BFF 接口改造  
- [ ] 创建 DashboardModule（nestjs-expert 规范）  
- [ ] 改造图表数据聚合接口（DTO 规范、响应格式统一）  
- [ ] 单元测试覆盖核心 Service 方法  
  
### Phase B: 前端页面实现  
- [ ] Dashboard 页面组件（react-expert 规范）  
  - [ ] 从 src/components/charts/ 引用图表组件  
  - [ ] 实现 loading/error/empty 三态  
- [ ] Analytics 页面组件...  
- [ ] Report 页面组件...  
  
### Phase C: 特性开关集成  
- [ ] FeatureFlag 接入，控制新旧版本切换

### 5.4 Stage 5：Reviewer（质量审查）

实施完成后，AI 加载 `react-reviewer` + `nestjs-reviewer`，生成评审报告：

## 代码评审报告 - 低码迁移 PR #142  
  
### CRITICAL (1) ❌ 必须修复  
1. analytics/index.tsx:87 - 直接引入 import * as echarts from 'echarts'  
   违反图表封装层规范，必须改为 src/components/charts/ 中的封装组件  
  
### HIGH (3) ⚠️ 强烈建议修复  
1. dashboard/DashboardPage.tsx:23 - loading 态未实现 Skeleton，使用了空白等待  
2. BFF dashboard.service.ts:45 - 未捕获下游 API 超时异常，高流量时会 500  
3. report/ReportChart.tsx:12 - 颜色值 '#3a7bd5' 硬编码，应使用 chartTheme.ts  
  
### MEDIUM (2) 建议修复  
...  
  
### LOW (1) 可选优化  
...

TL 看到 CRITICAL 有 1 项，要求开发者修复后重新提 PR，CRITICAL 清零后才合并。

---

## 6. 日常需求迭代规范（S/M/L 三档技能组合）

### 6.1 需求分级标准

![S/M/L 三档技能组合|697](file:///Users/guohaohao/Desktop/AI-joker/%E5%85%A8%E8%87%AA%E5%8A%A8AI%E5%BC%80%E5%8F%91%E6%B5%81%E7%A8%8B/adk-handbook/images/09_sml_skill_combination.svg?lastModify=1774875540)

|档位|时间估算|典型场景|技能组合|
|---|---|---|---|
|**S档**|< 30分钟|样式调整、文案修改、小交互|Tool Wrapper → 直接实现 → 人工自查|
|**M档**|30分钟~4小时|单功能模块、新组件、接口改造|Inversion → Generator → Tool Wrapper → 实现 → Reviewer|
|**L档**|> 4小时|跨模块、架构性改动、新功能模块|完整 feature-pipeline（含 TL 节点）|

### 6.2 S 档：快速执行流程

1. 明确对应技术栈（React? NestJS?）  
2. 手动激活对应 Tool Wrapper Skill  
3. AI 按规范实现  
4. 开发者人工自查 3 项：  
   □ 是否用了内联 style？  
   □ 是否有 any 类型？  
   □ 图表是否通过封装层引用？  
5. 提 PR（无需专项 Review，Code Owner 常规 Review）

**S 档注意事项**：即使是小改动，Tool Wrapper 的加载也不可省略，这是保证规范一致性的最后一道防线。

### 6.3 M 档：标准执行流程

1. 激活 requirement-intake + feature-pipeline（M档路径）  
2. Inversion 采访完成（3阶段）  
3. Generator 输出 spec.md  
   → TL 在 5分钟内轻量确认（不需要深度 Review，只确认范围没跑偏）  
4. Generator 输出 task-list.md  
5. 加载对应 Tool Wrapper Skill  
6. AI 按 task-list 实施  
7. Reviewer 生成评审报告  
8. 开发者自行修复 CRITICAL 和 HIGH  
9. 提 PR（附评审报告截图/链接）

**M 档时间控制**：Inversion + Generator 整个阶段控制在 15-20 分钟内，不要让前置流程占用过多时间。

### 6.4 L 档：Pipeline 全流程

1. 激活 feature-pipeline（L档完整路径）  
2. Inversion 深度采访（可能需要多轮）  
3. Generator 生成 spec.md + design.md  
   → TL 深度评审 design.md（30~60 分钟）  
   → 确认架构方案、评估风险  
   → Checkpoint 批准  
4. 拆解子 Track（每个 Track 独立走 M 档流程）  
   → 各 Track 并行开发  
   → 每个 Track 各自 Reviewer 评审  
5. TL 合并评审  
   → 查看所有 Track 的 Reviewer 报告汇总  
   → 重点检查跨 Track 的接口一致性  
6. 分批 PR 合并

**L 档团队协作**：建议在 design.md 确认后开一个 15 分钟的对齐会，确保所有参与开发者理解子 Track 分工和接口约定，减少后期整合成本。

### 6.5 代码 Prompt 最佳实践

**有效 Prompt 结构**：

[工作类型] + [技术上下文] + [具体任务] + [约束条件]  
  
示例（好）：  
"帮我在 src/pages/analytics/ 下创建一个新的 AnalyticsDashboard 组件。  
需要展示折线图（通过 src/components/charts/LineChart）和数据表格。  
数据从 /api/analytics/trend 接口获取，需要实现 loading/error/empty 三态。  
样式使用 Tailwind，不要 inline style。"  
  
示例（不好）：  
"帮我写一个分析页面的图表组件"

**触发 Inversion 的正确方式**：

"我有一个需求：[一句话描述]。请用 requirement-intake skill 来帮我整理需求。"

**调用 Reviewer 的正确方式**：

"请用 react-reviewer skill 评审以下代码：[粘贴代码或文件路径]"

### 6.6 并行开发协调规范

L档需求拆分为多个子 Track 并行时：

**接口契约先行**：在各 Track 开始实施前，先在 design.md 里确定所有 Track 间的接口契约（API 路径、参数、响应格式），避免后期整合时不兼容。

**共享 Skill 的版本一致性**：多人同时开发时，`.skills/` 下的技能包以 Git 主分支为准，任何 SKILL.md 的修改必须先合并主分支再生效，不允许在 feature 分支上修改技能包。

**Reviewer 时序**：子 Track 的 Reviewer 在各自 PR 阶段执行，跨 Track 的整合检查在最终 TL 合并评审时进行。

---

## 7. 团队分工与规范维护

### 7.1 角色与责任边界

![团队角色分工|697](file:///Users/guohaohao/Desktop/AI-joker/%E5%85%A8%E8%87%AA%E5%8A%A8AI%E5%BC%80%E5%8F%91%E6%B5%81%E7%A8%8B/adk-handbook/images/12_team_roles.svg?lastModify=1774875540)

#### Tech Lead（1-2人）

**技能包维护**：

- 每月评审一次 `.skills/` 下所有技能包，更新过时的规范
    
- 新库/框架接入时，及时新增 Tool Wrapper skill
    
- 根据 Reviewer 发现的高频问题，更新对应 checklist
    

**评审职责**：

- M 档：轻量确认 spec.md 范围（5 分钟内）；看 Reviewer 报告的 CRITICAL 项
    
- L 档：深度评审 design.md（30-60 分钟）；把关跨 Track 接口一致性；最终合并前的综合评审
    

**推广与培训**：

- 维护《模式选择指南》（本文档第 6 章）
    
- 新人入职 Onboarding：1 小时讲解 5 种模式 + 实操演示
    
- 每月 15 分钟的规范评审会（讨论哪些规范需要更新）
    

#### 开发者

**日常执行**：

- 按 S/M/L 分级选择技能组合
    
- M 档主动触发 Inversion，不跳过采访阶段
    
- 提 PR 前自行修复 Reviewer 报告的 CRITICAL 和 HIGH
    
- PR 描述中附上 Reviewer 报告链接
    

**反馈责任**：

- 发现 SKILL.md 有错误或遗漏 → 及时告知 TL，不要绕过规范
    
- 新接入的第三方库 → 申请 TL 新增 Tool Wrapper，不要自己临时封装
    
- 某类 Bug 反复出现 → 反馈给 TL 添加到 Reviewer checklist 的 CRITICAL 级别
    

### 7.2 SKILL.md 维护 SOP

**新增一个 Tool Wrapper Skill 的步骤**：

1. TL 确认需求（什么库、什么规范要封装）  
2. 创建目录：.skills/tool-wrappers/[lib-name]-expert/  
3. 编写 SKILL.md：  
   - frontmatter：name、description、triggers  
   - 目录结构规范（如果涉及文件组织）  
   - 强制规则（MUST/禁止 用语）  
   - 推荐写法示例  
4. 将参考文档放入 references/  
5. 在团队群里公告，附使用示例  
6. 两周后收集反馈，根据反馈迭代

**更新 Reviewer Checklist 的触发条件**：

- 同一类问题在 Reviewer 报告中连续 3 次出现 → 评估是否升级 Severity
    
- 发生过线上事故且可通过 Review 预防的 → 直接添加为 CRITICAL
    
- 新技术规范采用（如引入新的状态管理库）→ 更新对应 checklist
    

### 7.3 SKILL.md 示例模板（开箱即用）

**通用 SKILL.md 模板**：

---  
name: [skill-name]  
description: [一句话描述，AI 在什么情况下会激活此技能]  
triggers:  
  - "[触发关键词1]"  
  - "[触发关键词2]"  
---  
  
# [Skill 名称]  
  
## 你的角色  
[告诉 AI 在使用此技能时扮演的角色]  
  
## 适用范围  
[此技能适用于什么场景]  
  
## 强制规则（AI 必须遵守，使用 MUST/禁止 等强化措辞）  
- MUST：[规则1]  
- 禁止：[规则2]  
- MUST NOT：[规则3]  
  
## 目录结构（如适用）  
[代码块展示目录结构]  
  
## 推荐写法  
[代码示例，展示"好的写法" vs "不要这样写"]  
  
## 完成信号  
[告诉 AI 何时算完成了此技能的任务]

---

## 8. 成功指标与风险控制

### 8.1 可量化的成功指标

**代码质量**：

|指标|初始值（估算）|目标（3个月）|目标（6个月）|
|---|---|---|---|
|Reviewer CRITICAL 数/PR|5+|≤ 2|≤ 0.5|
|相同类型 Bug 复现率|高|降低 50%|降低 80%|
|any 类型使用数（全项目）|100+|≤ 30|≤ 10|

**开发效率**：

|指标|初始值|目标（3个月）|
|---|---|---|
|M档需求需求理解返工率|30%+|≤ 10%|
|PR 被打回需大改的比例|40%+|≤ 15%|
|新人能独立完成 M 档需求的时间|2周|1周|

**规范遵守**：

|指标|目标|
|---|---|
|图表组件通过封装层引用率|100%（Reviewer 强制检测）|
|M 档需求触发 Inversion 的比例|≥ 90%|
|SKILL.md 季度更新频率|TL 每月至少 1 次迭代|

### 8.2 风险识别与控制

**风险 1：AI 不遵守 SKILL.md 约束**

- 征兆：Reviewer 发现大量 CRITICAL，但开发者声称 AI 已经遵守规范
    
- 根因：SKILL.md 的指令不够明确，AI 产生歧义
    
- 应对：在 SKILL.md 中使用强化措辞（DO NOT、MUST NOT、absolutely forbidden），并附上明确的错误示例
    

**风险 2：Inversion 被开发者跳过**

- 征兆：M 档需求频繁出现需求理解偏差、返工
    
- 根因：流程压力下开发者嫌 Inversion 耗时
    
- 应对：TL 检查 PR 时确认是否有 spec.md（有 spec.md = Inversion 走过了）；返工成本纳入绩效考量
    

**风险 3：SKILL.md 规范过期失效**

- 征兆：Reviewer 不再能发现真正的问题，开发者认为 Reviewer 形同虚设
    
- 根因：技术栈升级但 SKILL.md 没有同步更新
    
- 应对：将"SKILL.md 评审"加入每月 Sprint 计划；引入新库必须先更新 SKILL.md 才能使用
    

**风险 4：Pipeline 在大团队中成为瓶颈**

- 征兆：TL 的 Checkpoint 评审积压，开发者等待时间过长
    
- 根因：TL 评审节点没有明确 SLA
    
- 应对：TL Checkpoint 评审 SLA：M档 2小时内，L档 1个工作日内；超时视为自动通过
    

**风险 5：不同 AI 工具的 Skill 兼容性问题**

- 征兆：同一 SKILL.md 在不同工具（Claude vs Cursor vs Gemini）上效果差异大
    
- 根因：各工具对 SKILL.md 格式和 frontmatter 的解析存在差异
    
- 应对：团队统一使用 Sonnet 4.6（已有共识）；测试 SKILL.md 时用同一工具验证
    

### 8.3 渐进落地路线图

第 1 个月：基础铺设  
  ✓ TL 编写 3 个 Tool Wrapper（react-expert / nestjs-expert / chart-expert）  
  ✓ TL 编写 react-reviewer（包含核心 checklist）  
  ✓ 全员培训：Tool Wrapper 用法 + Reviewer 用法  
  ✓ 所有 PR 开始要求附 Reviewer 报告  
  
第 2 个月：流程强化  
  ✓ 加入 requirement-intake（Inversion）  
  ✓ M 档需求开始强制触发 Inversion  
  ✓ 编写 spec-generator 和 task-generator  
  ✓ 收集首月 Reviewer 报告的高频问题，更新 checklist  
  
第 3 个月：全面闭环  
  ✓ feature-pipeline 上线（L 档需求强制走完整 Pipeline）  
  ✓ 评估第 1-2 月的指标数据  
  ✓ 根据数据调整 Severity 分级和 Checklist  
  ✓ 将 .skills/ 维护流程固化进团队 SOP  
  
持续迭代（3 个月后）  
  - 每月 SKILL.md 评审会  
  - 根据项目演进新增 Tool Wrapper  
  - 探索更多模式（如 Generator 扩展到自动生成测试用例）

---

## 附录 A：完整 react-reviewer checklist 模板

# React 代码评审 Checklist  
  
## CRITICAL（阻塞合并，必须修复）  
- [ ] 禁止直接 import ECharts，必须通过 src/components/charts/ 封装层  
- [ ] 禁止 any 类型出现在函数签名、Props、API 响应处理中  
- [ ] 所有异步操作必须有 catch / error 处理（Promise 不可裸露）  
- [ ] 禁止在生产代码中出现 console.log  
- [ ] 敏感信息（Token、密钥）禁止出现在前端代码中  
  
## HIGH（强烈建议在合并前修复）  
- [ ] 禁止 inline style（所有样式必须用 Tailwind 类名）  
- [ ] 图表颜色禁止硬编码，必须使用 src/theme/chartTheme.ts  
- [ ] useEffect 依赖数组不可遗漏（避免无限循环或陈旧闭包）  
- [ ] 组件 loading/error/empty 三态必须全部实现  
- [ ] useCallback/useMemo 的依赖数组必须准确  
  
## MEDIUM（建议修复，下轮迭代最迟处理）  
- [ ] 组件 Props 必须有完整 TypeScript 类型定义  
- [ ] 自定义 Hook 命名必须以 use 开头  
- [ ] 复杂逻辑（>15行）必须有注释说明意图  
- [ ] 组件单一职责：超过 200 行的组件考虑拆分  
  
## LOW（可选优化，不阻塞合并）  
- [ ] 可以用 const 枚举替代魔法字符串  
- [ ] 重复代码可抽取为 Hook 或工具函数  
- [ ] 注释更新是否同步（代码改了但注释没改）

---

## 附录 B：渐进式接入指南（现有项目）

对于已有代码库的项目，不需要从零改造。渐进接入步骤：

**Week 1-2：工具铺设（不改业务代码）**

1. 在项目根目录创建 `.skills/` 文件夹
    
2. TL 编写 `react-expert` skill（2-3小时）
    
3. 编写 `react-reviewer` skill + `react-checklist.md`（2-3小时）
    
4. 团队内部测试：新功能开发试用一次 Tool Wrapper + Reviewer
    

**Week 3-4：流程嵌入（只影响新需求）**

5. 新的 M 档需求开始触发 Inversion（现有需求不影响）
    
6. 所有 PR 开始附 Reviewer 报告（存量代码不审查，只看新增代码）
    
7. 收集反馈，更新 SKILL.md
    

**Month 2+：覆盖面扩大**

8. 逐步覆盖 NestJS 侧的 skill
    
9. 高频问题纳入 CRITICAL checklist
    
10. 引入 Pipeline 处理 L 档需求