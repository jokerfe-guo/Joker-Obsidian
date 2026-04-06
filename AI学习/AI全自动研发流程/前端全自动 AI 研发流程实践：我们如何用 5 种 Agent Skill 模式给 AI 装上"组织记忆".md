
> **作者**：郭豪豪 · 本地生活前端
> **背景**：React 18 + TypeScript + @kmi/react + qiankun 微前端技术栈 
> **关键词**：AI 工程化、Agent Skill、代码规范、研发流程

---

## 一、开篇：没有护栏的 AI，是最危险的效率工具

去年团队全面引入 AI 辅助编码，最初大家都很兴奋——写页面的速度确实快了不少。

但几个月后，我们开始意识到有些地方不太对劲。

代码 Review 的时候，同样是"商品列表页"，有人用了我们封装好的 `bizRequest`，有人直接写了裸 `fetch()`，还有人甚至 `import axios from 'axios'`——明明我们早就把 axios 剔出依赖了。问他为什么这么写，他说："AI 就这么生成的，我以为没问题。"

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

![[Gemini_Generated_Image_b1xo08b1xo08b1xo.png]]


传统方式里，规范靠人传人——入职培训、Wiki、Code Review 评论，这些都有效。但问题是 AI 不会参加入职培训，不会读 Wiki，Code Review 只能在代码写完后纠错，成本已经发生了。

AI 需要的是一种"在生成代码的那一刻就知道规范"的机制。

### 2.2 市面上的几种方案对比

#### Kiro（Amazon 推出的 Spec-first 方案）

Kiro 的理念是"先写规格，再让 AI 按规格生成代码"。你需要先在 `.kiro/specs/` 里写好功能规格，AI 按规格生成代码，Steering files 则用来约束 AI 的全局行为。

**优点**：规格文件驱动，需求可追溯；Steering 文件思路和 Tool Wrapper 很像。

**局限**：规格文件还是要人工手写，对于快速迭代的业务需求来说写规格的时间成本不低；没有内置代码质量审查；需要配套 Kiro IDE，工具绑定较强。

**适合场景**：有专职架构师、迭代节奏不算很快、需要严格需求文档管理的团队。

#### Superpower（Cursor Rules 增强方案）

本质上是通过 `.cursorrules` 或类似的规则文件给 AI 注入上下文，算是目前门槛最低的方案。

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

Google Agent Development Kit（ADK）提炼出的 5 种 Agent Skill 设计模式，我觉得这是目前把"AI + 团队规范"结合得最系统的方案。

先简单对比一下四种方案：

![[Gemini_Generated_Image_7gzcgf7gzcgf7gzc (1).png]]


我们不是因为 ADK 最完美才选它，而是因为它最匹配我们当前最痛的两个问题：**需求老是被 AI 误解**（→ Inversion 直接命中），**PR 里反复出现同类 CRITICAL 问题**（→ Reviewer 直接命中）。

---

## 三、5 种 Agent Skill 设计模式

在讲我们的具体实践之前，先把 5 种模式讲清楚。这是整套方案的理论基础。

![[Gemini_Generated_Image_qyqzikqyqzikqyqz.png]]

### 什么是 Agent Skill

一个 Skill 就是一个结构化的技能包，告诉 AI "在做这类任务时，你要遵守什么规则、用什么模板、参考什么文档"。

```
my-skill/  
├── SKILL.md       # AI 行为指令，核心文件  
├── assets/        # 输出模板（代码模板、文档模板）  
└── references/    # 参考资料（API 文档、checklist、规范说明）

```
`SKILL.md` 里的 frontmatter 定义技能的元信息，正文是给 AI 的行为约束：

---  
name: zustand-store-expert  
description: 当需要新建 Zustand Store 时激活  
triggers:  
  - "新建 Store"  
  - "create store"  
---  
​  
# Zustand Store 规范  
​  
## 强制规则  
- Store 必须拆成四个文件：create.ts / initial.ts / selectors.ts / index.ts  
- create.ts 禁止定义 interface，统一放 initial.ts  
- 新建公共 Store 必须追加到 barrel export  
...

和把所有规范堆在 system prompt 里相比，Skill 的优势是**按需加载**——写 Store 时加载 `zustand-store-expert`，写 Service 时加载 `biz-request-expert`，不相关的规范不占上下文，AI 的注意力更集中。

### 5 种模式速览

![[Gemini_Generated_Image_1ymxpb1ymxpb1ymx.png]]


---

**模式一：Tool Wrapper（工具包装器）**

把特定库或框架的使用规范装进独立的技能包。需要用到这个技术栈时，加载对应的 Wrapper；不需要的时候，不占上下文。

典型 Wrapper：`kmi-react-expert`（路由/主题/样式规范）、`zustand-store-expert`（四文件模式）、`biz-request-expert`（统一请求层）。

---

**模式二：Generator（文档/代码生成器）**

从模板出发，填空式生成结构化产物。把"格式定义"放进 `assets/template.md`，把"写作规范"放进 `references/style-guide.md`，AI 按模板逐项填空。

我们有页面骨架生成器、Store 四文件生成器、Service 函数生成器，还有 10 类规范文档生成器（代码规范/目录规范/接口规范/状态管理规范等）。

---

**模式三：Inversion（反转采访）**

反转传统的"用户提 prompt → AI 执行"，让 AI 先扮演采访者，收集完所有信息才开始动手。SKILL.md 里有明确的门控指令：

> **在以下所有阶段完成前，你不允许生成任何代码或方案。**

为什么需要这个模式？AI 最大的问题之一是太急于执行。开发者说"帮我做个商品筛选功能"，AI 立刻开始写代码，但可能完全误解了"筛选"的含义——等写完才发现，重写成本极高。

Inversion 分三阶段采访：业务上下文 → 技术范围 → 约束与验收。全部回答完才生成代码。

---

**模式四：Reviewer（代码审查员）**

把"检查什么"（checklist）和"如何检查"（SKILL.md 指令）分开，让 AI 按严重级别逐项评分，输出结构化报告：

- **CRITICAL**：阻塞合并，必须修复（架构违规、安全漏洞）
    
- **HIGH**：强烈建议在合并前修复（类型错误、性能隐患）
    
- **MEDIUM**：建议修复（命名不规范、缺少注释）
    
- **LOW**：优化建议，下次迭代处理
    

TL 只需关注 CRITICAL，开发者自行修复 CRITICAL + HIGH，大幅降低 Review 人力成本。

---

**模式五：Pipeline（工作流管道）**

把多个模式串联成有检查点的严格工作流，每个阶段有明确的进入条件和完成条件，AI 不能跳步骤：

![[Gemini_Generated_Image_dzj6sndzj6sndzj6.png]]
---

## 四、落地实践：ai-pilot-skills 体系

理论讲完了，说说我们怎么在 vulpix 项目里落地的。

![[Gemini_Generated_Image_v3hed4v3hed4v3he.png]]

我们把所有技能包放在项目根目录的 `.ai/ai-pilot-skills/` 下，随代码一起进 Git。任何团队成员都能看到、修改、提 PR 更新规范。

```
ai-pilot-skills/  
├── tool-wrappers/  
│   ├── kmi-react-expert/      # @kmi/react + qiankun 路由/样式规范  
│   ├── zustand-store-expert/  # Zustand 四文件模式规范  
│   └── biz-request-expert/    # 统一请求层 bizRequest 规范  
├── generators/  
│   ├── page-generator/        # 页面骨架生成（含 4 个模板文件）  
│   ├── store-generator/       # Store 四文件骨架  
│   ├── service-generator/     # Service 函数骨架  
│   └── ...（10 类规范文档生成器）  
├── inversions/  
│   └── requirement-interview/ # 需求访谈 + 技术文档生成  
├── reviewers/  
│   └── code-reviewer/         # 分级代码审查  
└── pipelines/  
    ├── auto-dispatch/         # 自动评估 S/M/L 复杂度  
    └── feature-pipeline/      # S/M/L 三级完整工作流
```

### 三条核心 Blocker 规范

从我们的 checklist 里，选 3 条最有代表性的 Blocker 级规范，说明为什么是 Blocker：

**规范一：所有新增请求必须用 `bizRequest`**

```
// ❌ 违规写法（CRITICAL）  
const data = await fetch('/api/items/list');  
import axios from 'axios';  
​  
// ✅ 合规写法  
import { bizRequest } from '@/utils/bizRequest';  
const data = await bizRequest({ url: '/api/items/list' });
```

为什么是 Blocker：`bizRequest` 内部封装了统一的错误拦截、鉴权 Token 注入、响应格式统一化。绕过它意味着这些能力全部缺失，是线上 Bug 的高发源。

---

**规范二：新增路由必须含微前端四字段**

```
// ❌ 违规写法（CRITICAL）  
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

```
为什么是 Blocker：vulpix 作为 qiankun 微前端子应用，这 4 个字段控制主应用的 Layout 组件是否渲染。缺少任意一个，子应用页面顶部/侧边会出现主应用的导航栏，导致双层 Header，布局完全错乱。这个问题在开发环境不会报错，只在集成到主应用后才能发现，修复成本高。

---

**规范三：Store 四文件职责严格分离**

```
reservationCapacity/  
├── create.ts    # 只含 zustand create，禁止定义 interface  
├── initial.ts   # interface 定义 + 初始值  
├── selectors.ts # 所有 selector 函数  
└── index.ts     # barrel export
```

为什么是 Blocker：早期有人把 interface 写在 `create.ts` 里，Zustand v5 的 TypeScript 推断机制会导致 `initial.ts` 里引用这些 interface 时出现循环引用，表现为类型推断失效，排查成本极高。现在 `create.ts` 禁止定义 interface 是铁律。

---

## 五、S/M/L 三档流程：不同体量走不同路径

我们把需求按改动规模分成三档，自动触发对应的工作流。

### 分级评估矩阵

![[Gemini_Generated_Image_1drrh31drrh31drr.png]]


自动调度器（`pipelines/auto-dispatch/`）会根据这个矩阵评估复杂度，在执行前输出：

📊 复杂度评估：M 级  
改动文件数：~5个 | 新建页面：1 | 新建 Store：1 | 新建 Service：1  
→ 进入 M 级流程：需求访谈 → 技术文档 → 代码骨架 → 代码审查

---

### S 级示例：修复筛选项 badge 计数不刷新

> 场景：商品列表页切换筛选条件后，顶部的已选筛选项 badge 数字不更新。

AI 执行流程：  
​  
![[Gemini_Generated_Image_hpqnsahpqnsahpqn.png]]

---

### M 级示例：新建"预约容量设置"页面

> 场景：业务方需要一个预约时段容量配置页面，包含按门店维度的容量列表和弹窗编辑功能。

![[Gemini_Generated_Image_78gf7e78gf7e78gf.png]]

**Stage 1 — 需求访谈（六维度，约 12 分钟）**

AI：这个预约容量是全局配置还是按某个维度管理？  
Dev：按门店维度，每个门店可以单独配置每个时段的容量上限。  
​  
AI：列表页需要哪些字段展示？支持分页吗？  
Dev：门店名称、时段（如 09:00-10:00）、容量上限、已预约数量。支持分页，每页 20 条。  
​  
AI：编辑操作是弹窗还是跳转详情页？需要批量编辑吗？  
Dev：弹窗编辑，暂不需要批量。  
​  
AI：容量值有无范围约束？提交接口是否已存在，还是需要新建 Service？  
Dev：1-9999，接口 POST /api/reservation/capacity/update 已有，需要新建 Service 函数。  
​  
AI：是否需要实时校验（容量不能低于已预约数量）？  
Dev：需要，提交时校验，低于已预约数量要给出提示。  
​  
AI：访谈完成，我来生成技术需求文档，请确认后继续。

**Stage 2 — 生成技术需求文档**（保存到 `docs/2026-03-30-reservation-capacity.md`）

文档包含：背景与目标 / 功能清单 / 数据流设计 / 接口定义 / 边界情况 / 验收标准 / 集成依赖 共 7 章节。

---

**Stage 3 — 生成代码骨架**

生成文件清单：  
``` tree
├── src/pages/ReservationCapacity/  
│   ├── index.tsx               # 页面主体（含列表 + 弹窗）  
│   └── index.module.less       # 样式文件（kebab-case CSS 变量，camelCase 引用）  
├── src/store/reservationCapacity/  
│   ├── create.ts               # Zustand store（不含 interface）  
│   ├── initial.ts              # interface 定义 + 初始值  
│   ├── selectors.ts            # selector 函数  
│   └── index.ts                # barrel export（已追加到全局 barrel）  
└── src/services/reservationCapacity/  
    ├── index.ts                # getCapacityList / updateCapacity 函数  
    └── type.ts                 # 接口类型定义
```

---

**Stage 4 — 代码审查报告**

```markdown
## Code Review Report — reservation-capacity

### CRITICAL (0) ✅

### HIGH (1) ⚠️
1. ReservationCapacity/index.tsx:Line 87
   EditModal 关闭时未清空 Store 的 editingItem 状态
   → 二次打开弹窗会显示上次编辑的数据
   修复建议：onClose 时 dispatch resetEditingItem()

### MEDIUM (2)
1. 容量校验逻辑（validateCapacity）写在组件内，建议抽到 utils/validation.ts
2. 弹窗的 Form 初始值直接用 editingItem，建议 cloneDeep 避免直接修改 store 引用

→ 请修复 HIGH 后提 PR，MEDIUM 可在下次迭代处理
```

耗时：约 1.5 小时（含访谈 12 min + 文档 5 min 确认 + 代码生成 50 min + 审查修复 25 min）

---

### L 级示例：商品批量分组管理功能

> 场景：新业务域，需要商品分组列表页 + 分组详情页 + 批量分配弹窗，涉及 2 个新页面、2 个新 Store、3 个新 Service。

![[Gemini_Generated_Image_ujjd2oujjd2oujjd 3.png]]

这是典型的 L 级需求，走完整 Pipeline：

```
Stage 1: 深度访谈（多轮，约 25 分钟）  
  → 输出：技术需求文档（docs/2026-03-xx-item-group.md）  
  → 门控节点：用户确认文档后才进 Stage 2  
​  
Stage 2: 完整代码骨架  
  生成顺序：Service 类型 → Service 函数 → Store → 页面 → 路由注册  
  每层生成后 AI 自检：  
  □ 请求是否用 bizRequest  
  □ 路由是否含四字段  
  □ Store 是否四文件分离  
​  
Stage 3: 代码审查  
  CRITICAL (2) ❌ 必须修复：  
    1. ItemGroupDetail/index.tsx:34  
       直接 import axios from 'axios'，违反 bizRequest 规范  
    2. ItemGroupList/route.ts  
       缺少 menuRender: false，微前端路由字段不完整  
  HIGH (1)：分组详情的删除操作缺少二次确认弹窗  
  → 修复 CRITICAL 后 CRITICAL = 0，可提 PR  
​  
Stage 4: 集成验证 Checklist  
  □ 路由已注册到 app.tsx ✅  
  □ Store 已追加到全局 barrel export ✅  
  □ 构建无报错 ✅  
  □ 微前端沙箱样式隔离验证 ✅  
  → 集成完成  
```
​  
耗时：约 3.5 小时（AI 主导，人工确认节点 3 次）

---

## 六、三个保证 AI 不出轨的关键设计

运行了两个月后，我们把效果最明显的三个机制单独拎出来说。

### 6.1 Inversion：从源头消灭需求误解

以前最让我头疼的不是代码 Bug，而是**开发完了才发现理解跑偏了**。

举个真实案例：某次需求是"支持商品批量操作"，AI 理解成了"批量选中然后统一提交"，但业务方的意思是"每次操作一次性作用到多个商品，不需要二次确认"——交互设计完全不同，做完推翻重来，两天工时打了水漂。

Inversion 的核心是**门控指令**：

在以下所有阶段完成前，禁止生成任何代码：  
​  
![[Gemini_Generated_Image_ryky3qryky3qryky.png]]

​  
✅ 三阶段全部完成后，生成技术需求文档，等用户确认后才开始写代码。

### 6.2 Blocker 分级 Review：把审查成本降到最低

在引入 Reviewer 之前，TL 做 Code Review 是最费时间的环节——每个 PR 要逐行看，经常一个 PR 要来回打 3-4 轮评论。

现在的流程是：

1. AI 先跑一遍 Reviewer，输出 CRITICAL/HIGH/MEDIUM/LOW 分级报告
    
2. 开发者自行修复所有 CRITICAL 和 HIGH，不 Pass 不能提 PR
    
3. TL 打开 PR 时，CRITICAL 已经是 0，只需确认关键的架构性问题，10-15 分钟搞定一个 PR
    

Reviewer checklist 的关键一点是**可以热换**。同一个 Reviewer Skill，把 `review-checklist.md` 换成 `security-checklist.md`，立刻变成安全审计工具；换成 `performance-checklist.md`，变成性能审计工具。

我们现在每次新发现一类高频问题，就把它加进 CRITICAL checklist，让 AI 在下一个 PR 开始主动检测。

### 6.3 Tool Wrapper：给 AI 装上"项目记忆"

以 `zustand-store-expert` 为例，对比一下有没有加载 Wrapper 的差异：

**不加载规范时（AI 默认生成）**：

```
// store/itemGroup.ts（AI 默认把所有东西塞一个文件）  
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
export interface ItemGroupState {  
  list: ItemGroup[];  
  loading: boolean;  
}  
export const initialItemGroupState: ItemGroupState = {  
  list: [], loading: false,  
};  
​  
// store/itemGroup/create.ts（只含 create，禁止 interface）  
import { create } from 'zustand';  
import { initialItemGroupState } from './initial';  
export const useItemGroupStore = create(() => ({ ...initialItemGroupState }));  
​  
// store/itemGroup/selectors.ts  
export const selectItemGroupList = (s: ItemGroupState) => s.list;  
export const selectItemGroupLoading = (s: ItemGroupState) => s.loading;  
​  
// store/itemGroup/index.ts（barrel export）  
export * from './create';  
export * from './selectors';  
export type { ItemGroup, ItemGroupState } from './initial';
```

两者的差别不只是代码风格，而是**可维护性**的本质差异。四文件模式下，interface 变更只改 `initial.ts`，不会触发 `create.ts` 的循环引用问题；selector 集中在 `selectors.ts`，组件层只 import selector，不直接操作 store 实现。

---

## 七、可量化的成效预期

以下是这套流程的目标值。

![[Gemini_Generated_Image_ed3spxed3spxed3s.png]]

---

### 对现有项目的渐进接入

规划按照“三阶段推进”执行：
- **第一阶段**完成最小闭环搭建，解决 AI 流程从 0 到 1 的问题
- **第二阶段**补齐前端规范与规则体系，解决流程可控性和一致性问题
- **第三阶段**完成团队推广，推动 AI 协作研发流程进入常态化执行

![[Gemini_Generated_Image_lqt413lqt413lqt4.png]]
---

## 九、结语：AI 最终需要"组织记忆"

回到开篇的问题：为什么引入 AI 之后代码反而乱了？

因为 AI 是一个无限能干但没有记忆的工具。它不知道你们团队三年来踩过的坑，不知道你们为什么选 Zustand 不选 Redux，不知道你们项目里有个 `bizRequest` 封装了所有鉴权逻辑。

解法不是让大家少用 AI，而是**把团队的记忆转移给 AI 的技能包**。

这件事有个很好的起点：从一个 Tool Wrapper 开始。把你们项目最常被 AI 写错的那个规范，比如请求层、路由约定、State 结构，写成一个 `SKILL.md`，花 2-3 小时，让 AI 明天就能用上。

先跑起来，然后根据 Reviewer 发现的高频问题持续更新，这个投入会快速回正。

---

## 附录：相关资源

- **Google ADK 官方文档**：[https://google.github.io/adk-docs/](https://google.github.io/adk-docs/)
    
- **5 种 Agent Skill 设计模式**：[Google Cloud Tech Twitter](https://x.com/GoogleCloudTech/status/2033953579824758855)
    
- **ai-pilot-skills 技能包**（vulpix 项目内）：`.ai/ai-pilot-skills/README.md`
    

---

_本文基于 vulpix 项目实战经验整理，虚拟示例均基于真实架构但不包含真实业务数据。_

_如果你们团队也在探索类似的方案，欢迎评论区交流。_
