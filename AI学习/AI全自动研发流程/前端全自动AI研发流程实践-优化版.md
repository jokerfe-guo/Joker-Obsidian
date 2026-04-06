# 前端全自动 AI 研发流程实践：我们如何用 5 种 Agent Skill 模式给 AI 装上"组织记忆"
---

## 一、开篇：没有护栏的 AI，是最危险的效率工具

去年团队全面引入 AI 辅助编码，最初大家都很兴奋——写页面的速度确实快了不少。

但三个月后，我们开始意识到有些地方不太对劲。

代码 Review 的时候，同样是"商品列表页"，有人用了我们封装好的 `bizRequest`，有人直接写了裸 `fetch()`，还有人甚至 `import axios from 'axios'`——明明我们早就把 axios 剥出依赖了。问他为什么这么写，他说："AI 就这么生成的，我以为没问题。"

类似的情况越来越多：

- 全项目的 `any` 类型从最初的 20 多处，半年时间涨到了近百处。因为 AI 生成的 Service 函数默认就喜欢用 `any`。
    
- 新人不知道新页面样式文件该命名成什么，AI 也不知道，有 `style.less`，有 `index.less`，有 `styles.module.less`，五花八门。
    
- 微前端子应用的路由需要 4 个特殊字段（`headerRender/footerRender/menuRender/menuHeaderRender: false`），缺少任意一个都会导致布局错乱。这个规则 TL 反复在群里提醒，但 AI 永远不知道，每次生成路由都要靠 Review 来补。
    

**根本原因只有一个**：AI 是没有"组织记忆"的工具。每次对话都是全新开始，它不知道你们团队的规范、你们项目的架构决策、你们过去踩过的坑。

后来我们意识到，解法不是禁止用 AI——而是**把团队规范写进 SKILL.md，让 AI 按需加载、按模式执行**。

这就是这篇文章要讲的事情。

---

## 二、我们选方案时踩过的弯路

在找到现在这套方案之前，我们调研了好几个路线，踩了不少坑，这里一并分享出来。

### 2.1 传统方式 vs AI 方式的本质差异

先说一下为什么传统研发流程的规范管控方式在 AI 时代开始失效。

<!-- 保留原图：传统开发模式 vs AI辅助模式 对比图 -->

|维度|传统开发模式|AI 辅助（无规范）|AI 辅助（有 Skill 体系）|
|---|---|---|---|
|需求理解|开发者读 PRD，自己理解|AI 直接猜需求，急于执行|Inversion 采访协议，先问完再动手|
|代码规范|依赖 Code Review 把关|AI 随意生成，不知道项目约定|Tool Wrapper 加载规范，生成即合规|
|技术文档|手动写，格式不统一|AI 乱写或跳过|Generator 模板填空，格式统一|
|质量把关|全靠人工逐行 Review|无质量门控，直接进主分支|Reviewer 分级报告，CRITICAL=0 才合并|
|新人上手|2 周熟悉规范|AI 帮倒忙，把坏习惯放大|1 周，AI 自带规范提示|

传统方式里，规范靠人传人——入职培训、Wiki、Code Review 评论，这些都有效。但问题是 AI 不会参加入职培训，不会读 Wiki，Code Review 只能在代码写完后纠错，成本已经发生了。

AI 需要的是一种"在生成代码的那一刻就知道规范"的机制。

### 2.2 市面上的几种方案对比

#### Kiro（Amazon 推出的 Spec-first 方案）

Kiro 的理念是"先写规格，再让 AI 按规格生成代码"。你需要先在 `.kiro/specs/` 里写好功能规格，AI 按规格生成代码，Steering files 则用来约束 AI 的全局行为。

**优点**：规格文件驱动，需求可追溯；Steering 文件思路和 Tool Wrapper 很像。

**局限**：规格文件还是要人工手写，对于快速迭代的业务需求来说写规格的时间成本不低；没有内置代码质量审查；需要配套 Kiro IDE，工具绑定较强。

**适合场景**：有专职架构师、迭代节奏不算很快、需要严格需求文档管理的团队。

#### Superpower（Cursor Rules 增强方案）

本质上是通过 `.cursorrules` 或类似的规则文件给 AI 注入上下文。

**优点**：接入成本极低，基本零改造；规则文件直接放在项目里就能生效。

**局限**：规则是静态全量注入，上下文很快被规则占满，AI 反而容易漏读某条规则；没有流程约束，AI 仍然会跳步骤、急于执行；只有规范，没有审查闭环。

我们试过这个方案大概三周，效果比没有规范好一点，但改善有限——我们的规范条目太多，规则文件越写越长，AI 开始"选择性遵守"。

**适合场景**：规范比较精简（不超过 20 条）、小团队快速起步的情况。

#### OpenSpec（规格驱动方案）

OpenSpec 的核心是 Delta Spec 机制：用 `ADDED/MODIFIED/REMOVED` 标记追踪每一次规格变更，`openspec/specs/` 里的文件始终描述系统的当前状态，类似"活文档"。

**优点**：系统状态完整可查，新人读完 specs/ 就能理解全局；历史变更精确可溯；兼容 21+ AI 工具。

**局限**：Delta Spec 的概念对普通前端开发者有学习成本；没有内置代码审查；需求采集仍需手动写 proposal.md；`/opsx:archive` 归档操作容易被遗忘；整体流程偏"重"。

我个人觉得 OpenSpec 在"系统文档化"这个维度做得很好，如果你们团队最大的痛点是"新人不知道系统长什么样"，值得考虑。但我们当时最紧迫的问题是"需求老被 AI 误解"和"PR 里反复出现同类问题"，OpenSpec 没有直接解决这两个痛点。

#### ADK Skill 模式（我们最终的选择）

Google Agent Development Kit (ADK) 官方博文提炼出的 5 种 Agent Skill 设计模式（参见 [Google Cloud Tech 推文](https://x.com/GoogleCloudTech/status/2033953579824758855)），我们在此基础上结合团队实际进行了落地实践，觉得这是目前把"AI + 团队规范"结合得最系统的方案。

> **说明**：ADK 原文给出的是通用的 Skill 设计模式框架，本文中的具体 Skill 实现（如 `zustand-store-expert`、`biz-request-expert` 等）和 S/M/L 分级流程是我们团队在此框架上的工程化实践，并非 ADK 官方内容。

<!-- 保留原图：四种方案对比图 -->

|维度|Kiro|Superpower|OpenSpec ADK|Skill 模式|
|---|---|---|---|---|
|需求采集|手动写 spec|无机制|手动写 proposal|Inversion 自动采访|
|规范管控|Steering files|静态规则注入|specs/ 主规格|Tool Wrapper 按需加载|
|文档生成|部分辅助|无|手动|Generator 模板驱动|
|代码审查|无内置|无内置|无内置|Reviewer 分级报告|
|工具绑定|Kiro IDE|Cursor|工具无关|工具无关|
|学习成本|中|低|中|中（模式名自解释）|
|渐进接入|较难|容易|较难|容易|

我们不是因为 ADK 最完美才选它，而是因为它最匹配我们当前最痛的两个问题：**需求老是被 AI 误解**（→ Inversion 直接命中）、**PR 里反复出现同类 CRITICAL 问题**（→ Reviewer 直接命中）。两个模式加进来，效果立竿见影。

**坦诚说一下这个方案的局限**：

- **Skill 编写有一定门槛**：写好一个高质量的 SKILL.md 需要对规范有深刻理解，不是随便写几条规则就够的。团队里通常需要 1-2 个人专门负责 Skill 质量。
    
- **模式组合需要经验**：5 种模式怎么组合、什么场景用什么模式，新手容易困惑。我们前期也走了弯路，比如给 S 级小需求也强制走完整 Pipeline，反而降低了效率。
    
- **AI 并非 100% 遵守**：即使有 Skill，AI 偶尔仍会"创造性发挥"，Reviewer 环节依然不可或缺。
    

---

## 三、5 种 Agent Skill 设计模式

在讲我们的具体实践之前，先把 5 种模式讲清楚。这是整套方案的理论基础。

<!-- 保留原图：AI 辅助研发的痛点与解决方案流程图 -->

### 什么是 Agent Skill

一个 Skill 就是一个结构化的技能包，告诉 AI "在做这类任务时，你要遵守什么规则、用什么模板、参考什么文档"。

my-skill/  
├── SKILL.md        # AI 行为指令，核心文件  
├── assets/         # 输出模板（代码模板、文档模板）  
└── references/     # 参考资料（API 文档、checklist、规范说明）

`SKILL.md` 里的 frontmatter 定义技能的元信息，正文是给 AI 的行为约束：

name: zustand-store-expert  
description: 当需要新建 Zustand Store 时激活  
triggers:  
  - "新建 Store"  
  - "create store"

和把所有规范堆在 system prompt 里相比，Skill 的优势是**按需加载**——写 Store 时加载 `zustand-store-expert`，写 Service 时加载 `biz-request-expert`，不相关的规范不占上下文，AI 的注意力更集中。

### 5 种模式速览

<!-- 保留原图：AI研发协作模式对比 -->

|模式|核心职责|解决的问题|一句话理解|
|---|---|---|---|
|**Tool Wrapper**|技术栈规范包，按需加载|AI 生成代码不符合项目规范|"AI 的技术栈说明书"|
|**Generator**|模板驱动文档/代码生成|文档格式不统一、代码骨架不规范|"AI 的填空题模板"|
|**Inversion**|AI 先采访再动手|AI 急于执行、误解需求|"AI 的记者采访流程"|
|**Reviewer**|分级代码审查报告|PR 质量参差不齐，人工 Review 成本高|"AI 的代码质检员"|
|**Pipeline**|多阶段检查点串联工作流|大需求 AI 跳步骤、失控|"AI 的流水线工序卡"|

**模式一：Tool Wrapper（工具包装器）**

把特定库或框架的使用规范装进独立的技能包。需要用到这个技术栈时，加载对应的 Wrapper；不需要的时候，不占上下文。

典型 Wrapper：`kmi-react-expert`（路由/主题/样式规范）、`zustand-store-expert`（四文件模式）、`biz-request-expert`（统一请求层）。

**模式二：Generator（文档/代码生成器）**

从模板出发，填空式生成结构化产物。把"格式定义"放进 `assets/template.md`，把"写作规范"放进 `references/style-guide.md`，AI 按模板逐项填空。

我们有页面骨架生成器、Store 四文件生成器、Service 函数生成器，还有 10 类规范文档生成器（代码规范/目录规范/接口规范/状态管理规范等）。

**模式三：Inversion（反转采访）**

反转传统的"用户提 prompt → AI 执行"，让 AI 先扮演采访者，收集完所有信息才开始动手。SKILL.md 里有明确的门控指令：

> **在以下所有阶段完成前，你不允许生成任何代码或方案。**

Inversion 分三阶段采访：业务上下文 → 技术范围 → 约束与验收。全部回答完才生成代码。

**模式四：Reviewer（代码审查员）**

把"检查什么"（checklist）和"如何检查"（SKILL.md 指令）分开，让 AI 按严重级别逐项评分，输出结构化报告：

- **CRITICAL**：阻塞合并，必须修复（架构违规、安全漏洞）
    
- **HIGH**：强烈建议在合并前修复（类型错误、性能隐患）
    
- **MEDIUM**：建议修复（命名不规范、缺少注释）
    
- **LOW**：优化建议，下次迭代处理
    

TL 只需关注 CRITICAL，开发者自行修复 CRITICAL + HIGH，大幅降低 Review 人力成本。

**模式五：Pipeline（工作流管道）**

把多个模式串联成有检查点的严格工作流，每个阶段有明确的进入条件和完成条件，AI 不能跳步骤：

<!-- 保留原图：5-STAGE PIPELINE 流程图 -->

---

## 四、落地实践：ai-pilot-skills 体系

理论讲完了，说说我们怎么在 vulpix 项目里落地的。

我们把所有技能包放在项目根目录的 `.ai/ai-pilot-skills/` 下，随代码一起进 Git。任何团队成员都能看到、修改、提 PR 更新规范。

ai-pilot-skills/  
├── tool-wrappers/  
│   ├── kmi-react-expert/       # @kmi/react + qiankun 路由/样式规范  
│   ├── zustand-store-expert/   # Zustand 四文件模式规范  
│   └── biz-request-expert/     # 统一请求层 bizRequest 规范  
├── generators/  
│   ├── page-generator/         # 页面骨架生成（含 4 个模板文件）  
│   ├── store-generator/        # Store 四文件骨架  
│   ├── service-generator/      # Service 函数骨架  
│   └── ... (10 类规范文档生成器)  
├── inversions/  
│   └── requirement-interview/  # 需求访谈 + 技术文档生成  
├── reviewers/  
│   └── code-reviewer/          # 分级代码审查  
└── pipelines/  
    ├── auto-dispatch/          # 自动评估 S/M/L 复杂度  
    └── feature-pipeline/       # S/M/L 三级完整工作流

### 三条核心 Blocker 规范

从我们的 checklist 里，选 3 条最有代表性的 Blocker 级规范，说明为什么是 Blocker：

**规范一：所有新增请求必须用 `bizRequest`**

// ❌ 违规写法 (CRITICAL)  
const data = await fetch('/api/items/list');  
import axios from 'axios';  
​  
// ✅ 合规写法  
import { bizRequest } from '@/utils/bizRequest';  
const data = await bizRequest({ url: '/api/items/list' });

为什么是 Blocker：`bizRequest` 内部封装了统一的错误拦截、鉴权 Token 注入、响应格式统一化。绕过它意味着这些能力全部缺失，是线上 Bug 的高发源。

**规范二：新增路由必须含微前端四字段**

// ❌ 违规写法 (CRITICAL)  
{ path: '/item/list', component: ItemList }  
​  
// ✅ 合规写法  
{  
  path: '/item/list',  
  component: ItemList,  
  headerRender: false,  
  footerRender: false,  
  menuRender: false,  
  menuHeaderRender: false,  
}

为什么是 Blocker：vulpix 作为 qiankun 微前端子应用，这 4 个字段控制主应用的 Layout 组件是否渲染。缺少任意一个，子应用页面顶部/侧边会出现主应用的导航栏，导致双层 Header，布局完全错乱。这个问题在开发环境不会报错，只在集成到主应用后才能发现，修复成本高。

**规范三：Store 四文件职责严格分离**

reservationCapacity/  
├── create.ts     # 只含 zustand create, 禁止定义 interface  
├── initial.ts    # interface 定义 + 初始值  
├── selectors.ts  # 所有 selector 函数  
└── index.ts      # barrel export

为什么是 Blocker：早期有人把 interface 写在 `create.ts` 里，Zustand v5 的 TypeScript 推断机制会导致 `initial.ts` 里引用这些 interface 时出现循环引用，表现为类型推断失效，排查成本极高。现在 `create.ts` 禁止定义 interface 是铁律。

---

## 五、S/M/L 三档流程：不同体量走不同路径

我们把需求按改动规模分成三档，自动触发对应的工作流。

### 分级评估矩阵

<!-- 保留原图：项目/任务复杂度评估维度表 -->

|维度|S 级（< 30min）|M 级（30min~4h）|L 级（> 4h）|
|---|---|---|---|
|改动文件数|1-2 个|3-8 个|9+ 个|
|新建页面|否|1 个|2+ 个|
|新建 Store|否|1 个|2+ 个|
|新建 Service 目录|否|1 个业务域|2+ 个业务域|
|路由变更|否|1 条|2+ 条|
|需求是否明确|明确|基本明确|模糊或复杂|

自动调度器（`pipelines/auto-dispatch/`）会根据这个矩阵评估复杂度，在执行前输出：

📊 复杂度评估：M 级  
改动文件数：~5个 | 新建页面：1 | 新建 Store：1 | 新建 Service：1  
→ 进入 M 级流程：需求访谈 → 技术文档 → 代码骨架 → 代码审查

### S 级示例：修复筛选项 badge 计数不刷新

> 场景：商品列表页切换筛选条件后，顶部的已筛选项 badge 数字不更新。

AI 执行流程：

1. **定位**：检索 `ItemListPage.tsx`，发现 `filterBadgeCount` 使用 `useMemo` 计算，依赖数组为 `[filterConfig]`，但实际依赖 `selectedFilters`（对象引用）
    
2. **修复**：将依赖数组改为 `[filterConfig, selectedFilters]`，由于 `selectedFilters` 是对象引用，额外用 `JSON.stringify` 做深比较
    
3. **代码审查（自动执行）**：CRITICAL (0) ✅ / HIGH (0) ✅ / MEDIUM (1) ⚠️ —— 建议将 `selectedFilters` 的比较逻辑抽成自定义 Hook → 可提交
    

<!-- 保留原图：AI 调试与修复执行流程 -->

**另一个 S 级示例：修改页面标题文案**

> 场景：产品要求把"商品管理"改成"商品中心"，涉及页面标题和面包屑。

AI 执行流程：

1. **定位**：搜索"商品管理"关键字，发现出现在 `route.ts` 的 `name` 字段和 `ItemManage/index.tsx` 的 `PageHeader` 组件中
    
2. **修复**：替换两处文案
    
3. **代码审查**：CRITICAL (0) ✅ —— 纯文案变更，无架构风险 → 直接提交
    

S 级需求的特点是：不走访谈、不生成文档，**只加载相关 Tool Wrapper + 自动 Review**，轻量快速。

### M 级示例：新建"预约容量设置"页面

> 场景：业务方需要一个预约时段容量配置页面，包含按门店维度的容量列表和弹窗编辑功能。

<!-- 保留原图：M级需求研发全流程 -->

**Stage 1 — 需求访谈（六维度，约 12 分钟）**

AI: 这个预约容量是全局配置还是按某个维度管理?  
Dev: 按门店维度，每个门店可以单独配置每个时段的容量上限。  
​  
AI: 列表页需要哪些字段展示? 支持分页吗?  
Dev: 门店名称、时段 (如 09:00-10:00)、容量上限、已预约数量。支持分页，每页 20 条。  
​  
AI: 编辑操作是弹窗还是跳转详情页? 需要批量编辑吗?  
Dev: 弹窗编辑，暂不需要批量。  
​  
AI: 容量值有无范围约束? 提交接口是否已存在，还是需要新建 Service?  
Dev: 1-9999，接口 POST /api/reservation/capacity/update 已有，需要新建 Service 函数。  
​  
AI: 是否需要实时校验（容量不能低于已预约数量）?  
Dev: 需要，提交时校验，低于已预约数量要给出提示。  
​  
AI: 访谈完成，我来生成技术需求文档，请确认后继续。

**Stage 2 — 生成技术需求文档**（保存到 `docs/2026-03-30-reservation-capacity.md`）

文档包含：背景与目标 / 功能清单 / 数据流设计 / 接口定义 / 边界情况 / 验收标准 / 集成依赖 共 7 章节。

**Stage 3 — 生成代码骨架**

├── src/pages/ReservationCapacity/  
│   ├── index.tsx               # 页面主体（含列表 + 弹窗）  
│   └── index.module.less       # 样式文件  
├── src/store/reservationCapacity/  
│   ├── create.ts               # Zustand store (不含 interface)  
│   ├── initial.ts              # interface 定义 + 初始值  
│   ├── selectors.ts            # selector 函数  
│   └── index.ts                # barrel export (已追加到全局 barrel)  
└── src/services/reservationCapacity/  
    ├── index.ts                # getCapacityList / updateCapacity 函数  
    └── type.ts                 # 接口类型定义

**Stage 4 — 代码审查报告**

### Code Review Report — reservation-capacity  
  
### CRITICAL (0) ✅  
  
### HIGH (1) ⚠️  
1. ReservationCapacity/index.tsx:Line 87  
   EditModal 关闭时未清空 Store 的 editingItem 状态  
   → 二次打开弹窗会显示上次编辑的数据  
   修复建议: onClose 时 dispatch resetEditingItem()  
  
### MEDIUM (2)  
1. 容量校验逻辑 (validateCapacity) 写在组件内, 建议抽到 utils/validation.ts  
2. 弹窗的 Form 初始值直接使用 editingItem, 建议 cloneDeep 避免直接修改 store 引用  
  
→ 请修复 HIGH 后提 PR, MEDIUM 可在下次迭代处理

耗时：约 1.5 小时（含访谈 12 min + 文档 5 min 确认 + 代码生成 50 min + 审查修复 25 min）

### L 级示例：商品批量分组管理功能

> 场景：新业务域，需要商品分组列表页 + 分组详情页 + 批量分配弹窗，涉及 2 个新页面、2 个新 Store、3 个新 Service。

<!-- 保留原图：L 级需求完整流水线 -->

这是典型的 L 级需求，走完整 Pipeline：

**Stage 1：深度访谈**（多轮, 约 25 分钟）→ 输出技术需求文档，用户确认文档后才进 Stage 2

**Stage 2：完整代码骨架**。生成顺序：Service 类型 → Service 函数 → Store → 页面 → 路由注册。每层生成后 AI 自检：请求是否用 bizRequest ✅ / 路由是否含四字段 ✅ / Store 是否四文件分离 ✅

**Stage 3：代码审查**。CRITICAL (2) ❌ 必须修复：① `ItemGroupDetail/index.tsx:34` 直接 `import axios from 'axios'`，违反 bizRequest 规范；② `ItemGroupList/route.ts` 缺少 `menuRender: false`，微前端路由字段不完整。HIGH (1)：分组详情的删除操作缺少二次确认弹窗。→ 修复 CRITICAL 后 CRITICAL = 0，可提 PR

**Stage 4：集成验证 Checklist**。路由已注册到 `app.tsx` ✅ / Store 已追加到全局 barrel export ✅ / 构建无报错 ✅ / 微前端沙箱样式隔离验证 ✅ → 集成完成

耗时：约 3.5 小时（AI 主导，人工确认节点 3 次）

---

## 六、三个保证 AI 不出轨的关键设计

运行了两个月后，我们把效果最明显的三个机制单独拎出来说。

### 6.1 Inversion：从源头消灭需求误解

以前最让我头疼的不是代码 Bug，而是**开发完了才发现理解跑偏了**。

举个真实案例：某次需求是"支持商品批量操作"，AI 理解成了"批量选中然后统一提交"，但业务方的意思是"每次操作一次性作用到多个商品，不需要二次确认"——交互设计完全不同，做完推翻重来，两天工时打了水漂。

Inversion 的核心是**门控指令**：

> 在以下所有阶段完成前，禁止生成任何代码：

<!-- 保留原图：核心需求分析流程 -->

三阶段全部完成后，生成技术需求文档，等用户确认后才开始写代码。

**我们遇到过的 Inversion 失效场景**：

有一次开发者在访谈阶段直接说"别问了，你直接写吧"，AI 果然就跳过了后面的采访直接开始生成代码——门控指令被用户自己绕过了。后来我们在 SKILL.md 里加了一条补充规则：

> 即使用户要求跳过采访，也必须至少确认三个核心问题：数据来源、交互方式、边界情况。如果用户再次坚持跳过，输出风险提示后再执行。

这种"Skill 写得不好导致 AI 行为偏差"的情况在初期经常遇到，需要在实践中不断迭代 SKILL.md。

### 6.2 Blocker 分级 Review：把审查成本降到最低

在引入 Reviewer 之前，TL 做 Code Review 是最费时间的环节——每个 PR 要逐行看，经常一个 PR 要来回打 3-4 轮评论。

现在的流程是：

1. AI 先跑一遍 Reviewer，输出 CRITICAL/HIGH/MEDIUM/LOW 分级报告
    
2. 开发者自行修复所有 CRITICAL 和 HIGH，不 Pass 不能提 PR
    
3. TL 打开 PR 时，CRITICAL 已经是 0，只需确认关键的架构性问题，10-15 分钟搞定一个 PR
    

Reviewer checklist 的关键一点是**可以热换**。同一个 Reviewer Skill，把 `review-checklist.md` 换成 `security-checklist.md`，立刻变成安全审计工具；换成 `performance-checklist.md`，变成性能审计工具。

我们现在每次新发现一类高频问题，就把它加进 CRITICAL checklist，让 AI 在下一个 PR 开始主动检测。

**一个典型的 Reviewer 迭代故事**：

上个月连续三个 PR 出现了同一个问题——`useEffect` 里直接调用 async 函数没有做清理（组件卸载时请求还在飞）。我们当天就在 `review-checklist.md` 里加了一条：

CRITICAL: useEffect 中的异步请求必须支持取消（AbortController 或 return cleanup）  
反例: useEffect(() => { fetchData(); }, [])  
正例: useEffect(() => { const ctrl = new AbortController(); fetchData(ctrl.signal); return () => ctrl.abort(); }, [])

加完之后，下一个 PR 里 AI 就自动把这个问题标记为 CRITICAL 了。这种"发现问题 → 更新 checklist → 自动检测"的闭环，是整套体系最让我兴奋的部分。

### 6.3 Tool Wrapper：给 AI 装上"项目记忆"

以 `zustand-store-expert` 为例，对比一下有没有加载 Wrapper 的差异：

**不加载规范时（AI 默认生成）**：
```
// store/itemGroup.ts (AI 默认把所有东西塞一个文件)  
interface ItemGroup { id: string; name: string; }  
interface ItemGroupState {  
  list: ItemGroup[];  
  loading: boolean;  
  setList: (list: ItemGroup[]) => void;  
}  
export const useItemGroupStore = create<ItemGroupState>()((set) => ({  
  list: [], loading: false,  
  setList: (list) => set({ list }),  
}));

**加载 zustand-store-expert 后（按规范生成）**：

// store/itemGroup/initial.ts  
export interface ItemGroup { id: string; name: string; }  
export interface ItemGroupState { list: ItemGroup[]; loading: boolean; }  
export const initialItemGroupState: ItemGroupState = { list: [], loading: false };  
  
// store/itemGroup/create.ts (只含 create, 禁止 interface)  
import { create } from 'zustand';  
import { initialItemGroupState } from './initial';  
export const useItemGroupStore = create(() => ({ ...initialItemGroupState }));  
  
// store/itemGroup/selectors.ts  
export const selectItemGroupList = (s: ItemGroupState) => s.list;  
export const selectItemGroupLoading = (s: ItemGroupState) => s.loading;  
  
// store/itemGroup/index.ts (barrel export)  
export * from './create';  
export * from './selectors';  
export type { ItemGroup, ItemGroupState } from './initial';
```


两者差别不只是代码风格，而是**可维护性**的本质差异。四文件模式下，interface 变更只改 `initial.ts`，不会触发 `create.ts` 的循环引用问题；selector 集中在 `selectors.ts`，组件层只 import selector，不直接操作 store 实现。

---

## 七、Skill 的维护与迭代机制

Skill 写好不是终点，**持续维护才是关键**。我们建立了一套轻量的迭代机制：

### 谁来维护

每个 Skill 目录下有一个 `OWNERS` 文件（借鉴 Google 的 OWNERS 机制），标注 1-2 个负责人。修改 Skill 的 PR 必须由 OWNER review。

### 怎么发现需要更新

三个信号触发 Skill 更新：

1. **Reviewer 高频重复**：同一类 CRITICAL/HIGH 问题在两周内出现 3 次以上 → 要么 Skill 规则不够明确，要么缺少对应的 checklist 条目
    
2. **开发者反馈**：有人在 PR 评论里说"AI 没遵守 XX 规范" → 检查对应 Skill 是否覆盖了这条规范
    
3. **技术栈升级**：比如 Zustand 从 v4 升到 v5 时，Store 的写法变了 → 对应 Tool Wrapper 必须同步更新
    

### 迭代流程

和代码一样走 PR 流程：提交修改 → OWNER review → 合并到主分支。每次修改需要在 PR 描述里说明"为什么改"和"预期效果"。

---

## 八、踩坑记录：Skill 写不好时会发生什么

说了这么多成功路径，也分享几个我们踩过的坑，帮你们少走弯路。

### 坑一：规则太模糊，AI "创造性解读"

早期我们的 `biz-request-expert` 里写的是：

> 请求应该使用 bizRequest 进行封装。

"应该"两个字给了 AI 太大的解读空间。有的场景 AI 觉得"这个请求比较简单，不需要封装"，就直接用了 fetch。后来改成：

> **必须**使用 bizRequest。任何新增的 HTTP 请求，无论 GET/POST/PUT/DELETE，禁止直接使用 fetch/axios/XMLHttpRequest。违反此规则标记为 CRITICAL。

规则要**绝对明确**，不要给 AI 留"酌情处理"的空间。

### 坑二：规则互相冲突，AI 无所适从

有段时间 `page-generator` 的模板里写了"每个页面一个 store 文件"，但 `zustand-store-expert` 要求"store 必须拆成四个文件"。AI 两个 Skill 都加载了，就开始"折中"——生成了两个文件的 store。两头不靠。

**教训**：多个 Skill 之间必须保持一致性。我们现在每新增或修改一个 Skill，都会跑一遍"冲突检查"，看是否和已有 Skill 矛盾。

### 坑三：给小需求上大流程，效率反降

前期我们没有 S/M/L 分级，所有需求都走完整 Pipeline。结果一个改文案的 S 级需求也要走"访谈 → 文档 → 骨架 → 审查"四个阶段，15 分钟能搞定的事情拖到了 40 分钟。

**教训**：流程的粒度要匹配需求的体量。后来加了 auto-dispatch，S 级需求自动跳过访谈和文档阶段，效率立刻回来了。

---

## 九、可量化的成效预期

我想诚实地说：这套流程我们还在运行中，以下是我们设定的目标值，部分已经达到，部分还在验证。

<!-- 保留原图：指标引入前后对比 -->

|指标|引入前（估算）|目标（3个月）|当前状态|
|---|---|---|---|
|Reviewer CRITICAL 数/PR|5+|≤ 2|✅ 已达成，近两周均值 1.2|
|M 档需求理解返工率|30%+|≤ 10%|📉 下降中，当前约 15%|
|PR 被打回大改比例|40%+|≤ 15%|📉 下降中，当前约 20%|
|新人独立完成 M 档时间|2 周|1 周|✅ 最近一位新人用了 6 天|
|bizRequest/路由规范遵守率|< 60%|100%|✅ 已达成|

最让我意外的是 bizRequest 规范遵守率的变化。以前每次 Review 都会发现裸 fetch 或 axios，现在几乎消失了——不是因为大家突然都记住了，而是 AI 生成的代码天然就符合规范，人就算想"偷懒"，AI 也不会配合他。

---

## 十、对现有项目的渐进接入

### 团队推广中的阻力与应对

坦白说，推广初期并不是所有人都买账。主要的阻力有两种：

**"写 Skill 是额外负担"**——这是最常见的反对声音。我们的应对方式是：不要求所有人写 Skill，而是由 TL 和 1-2 个核心开发先写好最痛的 3 个 Tool Wrapper（bizRequest、路由规范、Store 规范），其他人只需要"用"就行。当大家发现 AI 生成的代码不用再反复改的时候，抵触就自然消退了。

**"我不信 AI Review"**——有经验的开发者对 AI 审查质量持怀疑态度。我们做了两周的并行实验：人工 Review 和 AI Review 同时跑，统计 AI 的漏检率和误报率。结果 AI 在 CRITICAL 级别的检出率达到 92%（人工是 95%），误报率 8%——够用了。关键不是取代人工 Review，而是让人工 Review 聚焦在 AI 搞不定的架构决策上。

### 三阶段推进计划

当前规划按照"三阶段推进"执行：

**第一阶段（1 周）**：完成最小闭环搭建，解决 AI 流程从 0 到 1 的问题。搭建 `.ai/` 基础目录，写 3 个 Tool Wrapper + 1 个 Reviewer，跑通一个完整的 M 级需求。

**第二阶段（3 天）**：补齐前端规范与规则体系，解决流程可控性和一致性问题。补充剩余的 Generator、Inversion 等 Skill，升级 Reviewer checklist 覆盖全量规范。

**第三阶段（2 天）**：完成团队推广，推动 AI 协作研发流程进入常态化执行。明确新需求 AI 流程对齐，案例团队分享，收集团队反馈持续优化。

<!-- 保留原图：AI 协作研发流程落地推进时间线 -->

---

## 十一、结语：AI 最终需要"组织记忆"

回到开篇的问题：为什么引入 AI 之后代码反而乱了？

因为 AI 是一个无限能干但没有记忆的工具。它不知道你们团队三年来踩过的坑，不知道你们为什么选 Zustand 不选 Redux，不知道你们项目里有个 `bizRequest` 封装了所有鉴权逻辑。

解法不是让大家少用 AI，而是**把团队的记忆转移给 AI 的技能包**。

**如果你今天就想开始，可以做这三件事**：

1. **找到你们项目最常被 AI 写错的那个规范**（比如请求层、路由约定、State 结构），花 2-3 小时写成一个 `SKILL.md`，明天 AI 就能用上。
    
2. **写一个 Reviewer checklist**，把你最近 10 个 PR 里反复出现的问题列成 CRITICAL 条目，让 AI 帮你自动检测。
    
3. **先跑起来，然后根据 Reviewer 发现的高频问题持续更新**。这个投入会快速回正。
    

不要追求一步到位。从一个 Tool Wrapper 开始，让团队看到效果，然后自然地扩展到 Generator、Inversion、Pipeline。

### 附录：相关资源

- **Google ADK 官方文档**：[https://google.github.io/adk-docs/](https://google.github.io/adk-docs/)
    
- **5 种 Agent Skill 设计模式**（Google Cloud Tech 推文）：[https://x.com/GoogleCloudTech/status/2033953579824758855](https://x.com/GoogleCloudTech/status/2033953579824758855)
    
- **ai-pilot-skills 技能包**（vulpix 项目内）：`.ai/ai-pilot-skills/README.md`
    

---

_本文基于 vulpix 项目实战经验整理，虚拟示例均基于真实架构但不包含真实业务数据。_

_如果你们团队也在探索类似的方案，欢迎评论区交流。_