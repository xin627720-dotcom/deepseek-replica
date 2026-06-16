# 《修仙模拟器》(cultivation-simulator) 源码分析

> 分析对象：https://github.com/YoungJack/cultivation-simulator
> 分析日期：2026-06-16

## 一、项目定位

一个「现实习惯养成 × 修仙叙事」的轻量 Web 游戏：把现实中的**学习 / 运动 / 早睡 / 冥想**打卡，转化为《凡人修仙传》风格修仙世界里的「悟道 / 锻体 / 静修 / 打坐」，由 AI（Claude）实时生成修炼叙事，配合境界突破、随机奇遇等玩法形成正反馈循环。本质是一个带剧情包装的 habit-tracker。

口号：「把今天的努力，变成修仙世界的修为」。

## 二、技术栈

| 层 | 选型 |
|---|---|
| 框架 | **Next.js 16.2.9**（App Router，RSC）+ React 19 |
| 语言 | TypeScript 5 |
| 样式 | Tailwind CSS 4 + shadcn（`base-nova` 风格）+ `@base-ui/react` + framer-motion |
| 图标/通知 | lucide-react、sonner（toast） |
| ORM/DB | **Prisma 7** + `@prisma/adapter-libsql`，本地 SQLite 文件、生产 Turso/libSQL |
| AI | `@anthropic-ai/sdk`，模型 `claude-sonnet-4-6` |
| 认证 | 依赖里有 `next-auth@5 beta`，但**实际未使用**，自实现了一套简易账号体系 |

`next.config.ts` 将 `@prisma/adapter-libsql`、`@libsql/client` 标记为 `serverExternalPackages`（libSQL adapter 在服务端运行的必要配置）。

## 三、目录结构

```
src/
├── app/
│   ├── page.tsx            # 落地页（三步引导 + 特色卡片）
│   ├── login/page.tsx      # 登录
│   ├── create/page.tsx     # 创建角色：道号 + 密码 + 灵根测试问卷
│   ├── dashboard/page.tsx  # 核心面板（767 行，全部逻辑集中于此）
│   └── api/
│       ├── auth/login/route.ts   # POST 注册 / PUT 登录（scrypt 加盐）
│       ├── cultivator/route.ts   # POST 创建 / GET 查询修炼者
│       ├── tasks/route.ts        # POST 建任务 / PATCH 完成 / GET 列表
│       ├── narrative/route.ts    # POST 生成叙事 + 处理突破/奇遇
│       └── encounter/route.ts    # GET 触发奇遇 / POST 结算选项
├── lib/
│   ├── cultivation-data.ts # 灵根/境界/任务/NPC 数据 + 修炼计算函数
│   ├── encounter-data.ts   # 奇遇事件池 + 触发/结算逻辑
│   ├── narrative.ts         # AI 叙事引擎（含 25 套降级模板）
│   ├── prisma.ts            # Prisma 单例（libSQL adapter）
│   └── index.ts             # 仅导出游戏数据（客户端安全，不含 Prisma）
└── components/ui/           # shadcn 组件
prisma/schema.prisma         # 5 张表
```

## 四、数据模型（`prisma/schema.prisma`）

- **User**：账号名（唯一，登录用）、`password`（`salt:hash` 格式）、可选 email/bilibiliId/avatar。1:1 关联 Cultivator，1:N 关联 DailyTask。
- **Cultivator**：道号、灵根、境界 `realm` + 层数 `realmLevel`、当前修炼值 `cultivationExp`、累计 `totalExp`、`stamina`、`breakthroughCount`、`title`。1:N 关联 GameEvent。
- **DailyTask**：任务类型、是否完成、修炼值加成、日期。
- **GameEvent**：事件流水（日常修炼 / 突破 / 奇遇），含 `narrative`、`choices`(JSON)、`chosenOption`、`reward`(JSON)。
- **ShareCard**：分享卡（schema 中存在，但**代码中无任何读写**，属预留/未实现）。

## 五、核心玩法机制

### 1. 灵根系统（`cultivation-data.ts`）
6 档灵根（杂/四/三/双/异/天），稀有度 1–5，对应修炼速度倍率 1.0–1.5。
创建角色时通过 3 道问卷题积累五行元素，`determineSpiritualRoot` 按**出现的不同元素数量**判定：1 种→天灵根、5 种→杂灵根、4→四、3→三，2 种则 15% 概率变异为异灵根、否则双灵根。

> 注意：这套规则下「双灵根」是稀有度 3，而「天灵根」(稀有度5) 反而最容易出现（只要 3 题都选同一元素）。问卷设计与稀有度直觉略有偏差。

### 2. 境界体系
9 大境界（炼气→筑基→结丹→元婴→化神→炼虚→合体→大乘→渡劫），每境界有层数、第 1 层所需修炼值、每层递增量。突破计算：
- `getRequiredExp = expRequired + (level-1) * expIncrement`
- `canBreakthrough`：当前修炼值 ≥ 所需即可（满层时检查跨大境界）
- `performBreakthrough`：突破后**保留溢出修炼值**，跨大境界则层数归 1。

### 3. 每日任务与上限（`tasks/route.ts`）
5 种任务各有 `baseExp` 与 `dailyMax`（悟道3/锻体2/静修1/打坐2/历练2）。完成任务时按 `baseExp × 灵根倍率` 取整发放修炼值，并在事务中同步更新 `cultivationExp`/`totalExp`。完成后**有 30% 概率触发随机奇遇**。

### 4. 随机奇遇（`encounter-data.ts` + `encounter/route.ts`）
- 事件池目前 **3 个**手写奇遇（古洞府、灵狐渡劫、邪修伏击），每个 3 选项（低/中/高风险），加权随机抽取，每日上限 3 次。
- 触发：30% 概率（`shouldTriggerEncounter`）。
- 结算：低/中风险直接给奖励；**高风险**走概率判定 `resolveHighRiskOutcome`：基础成功率 40% + 灵根稀有度加成 + 境界加成，上限 85%；失败则把修炼值奖励反转为扣减。
- `serializeEncounter` 会**剥离 `successNarrative`** 再返回前端，防止提前剧透结果——这是个不错的服务端权威设计。

### 5. AI 叙事引擎（`narrative.ts`）
- 统一 `WORLD_SYSTEM_PROMPT` 设定仙侠文风 + 要求返回合法 JSON。
- 提供日常修炼 / 境界突破 / 随机奇遇 / NPC 对话四类生成函数，均用正则 `\{[\s\S]*\}` 从回复中抽取 JSON 解析。
- **降级方案很扎实**：API 失败时回落到 25 套本地模板（5 种修炼 × 5 变体），并按灵根映射 5 种文风（烈/柔/锐/韧/稳），`pickTemplate` 还做了「避免连续重复上一条」处理。客户端体验在无 key / 超时下不至于崩。
- 客户端密钥延迟初始化（`getClient`），避免 build 时缺 `ANTHROPIC_API_KEY` 报错。

### 6. 账号体系（`auth/login/route.ts`）
- 注册 POST / 登录 PUT，密码用 **Node `crypto.scryptSync` 加盐**（注释写的是 bcrypt，但实现其实是 scrypt），存储为 `salt:hash`。
- 前端登录态：仅把 `userId` 存进 `localStorage`，后续所有请求把 userId 当普通参数传给后端。

## 六、前端交互亮点

- `dashboard/page.tsx` 是整个应用的心脏，集中了状态加载、任务、突破、奇遇、历史。
- **乐观更新**：完成任务时先在前端预估并加上修炼值/标记完成，请求失败再回滚，成功后用服务端真值修正——交互很跟手。
- 突破契机以 toast + action 按钮提示；大境界突破有特效文案与 emoji 体系（按境界切换 ⚪🟢🟡💎🔥🌌💫🌟👑）。
- 落地页明确写了「⚠️ 真的去做再点。靠自觉——骗系统没意义，你骗不了自己」，把「无强校验」设计成自律契约而非漏洞，定位诚实。

## 七、值得关注的问题与风险

按严重程度排序：

### 🔴 1. 无真正鉴权 —— 越权访问（IDOR）
所有 API 仅凭请求里携带的 `userId`（来自 localStorage）识别身份，**服务端从不校验调用者是否拥有该 userId**。任何人只要拿到/猜到别人的 cuid，即可：
- `GET /api/cultivator?userId=别人` 读取其全部数据；
- `PATCH /api/tasks`、`POST /api/narrative`（突破）、`POST /api/encounter` 修改别人的角色。

依赖里装了 `next-auth` 却未启用，是最该补的安全短板（应改用服务端 session / JWT，从会话而非请求体取 userId）。

### 🟠 2. 两条互相冲突的用户创建路径
- `/api/auth/login` POST：带密码创建（正式路径）。
- `/api/cultivator` POST：**不带密码**创建 User + Cultivator。

后者会产生 `password = null` 的账号，且与 schema 注释「新账号必填密码」矛盾。`cultivator` 路由的 POST 实质是历史遗留/死代码，建议删除，统一走 auth 路径。

### 🟠 3. 两套并存且不一致的「奇遇」实现
- 真正在用的是 `encounter-data.ts`（数据驱动、有风险判定、服务端结算）。
- 但 `narrative/route.ts` 里还有一条 `case "ENCOUNTER"`：每次调用都让 AI 现编一个奇遇，选项只按 risk 发放**固定** 15/30/50 修炼值，**没有成功率判定、不计入每日上限**，且写入的事件类型是 `RANDOM_ENCOUNTER`（与限额统计用的 `ENCOUNTER` 不同表口径）。这条路径 dashboard 已不调用，属应清理的并行实现，留着易混淆。

### 🟡 4. `stamina`（灵力）是「死机制」
schema、UI（显示「灵力 x/100，每日重置」）都有它，但**全代码无任何消耗或重置 stamina 的逻辑**，永远是 100。要么补全（任务/突破消耗 + 每日 cron 重置），要么从 UI 移除以免误导。

### 🟡 5. 高风险失败文案 bug
`encounter/route.ts` 失败分支：
```ts
label: "未能" + r.label.replace("获得", "获得") + "……"
```
`replace("获得","获得")` 是空操作，显然是复制粘贴笔误，本意应是改写措辞（如替换为「错失」）。

### 🟡 6. 降级模板的防重复状态是跨用户的全局变量
`narrative.ts` 的 `lastUsedIndex` 是模块级对象。在 serverless / 多实例部署下既不可靠，又会在所有用户间共享（A 用户的上一次选择影响 B）。无害但不严谨，宜下放到请求级或随机化。

### 🟡 7. 时区与「每日」边界
所有「今日」判断用服务端本地时间 `setHours(0,0,0,0)`。每日上限/重置按**服务器本地午夜**计算，部署在 UTC 时对中国用户会在早上 8 点重置，体验上需注意。

### ⚪ 8. 其他
- `ShareCard` 表完全未实现（B 站分享卡是产品规划但无代码）。
- `getEncounterById(gameEvent.title)` 用 title 去匹配 id，必然首查失败再走 title 回退，能跑通（标题唯一）但语义错位。
- 无任何自动化测试（`test-*.js` 被 `.gitignore` 排除，仓库内无测试）。
- `.gitignore` 显示项目还有视频制作、软著材料、mobile 端等未公开资产，说明这是个有运营/商业化打算的实际项目。

## 八、总体评价

**优点**：技术选型新颖统一（Next 16 + Prisma 7 + libSQL + Claude），玩法闭环完整且立意巧妙（自律工具的游戏化外壳很自洽），AI 叙事有**完善的本地降级**，前端乐观更新体验流畅，奇遇系统的服务端权威/防剧透设计专业。代码组织清晰，`lib` 与 Prisma 的客户端/服务端边界划分得当。

**主要短板**：**安全鉴权缺失（IDOR）是头号问题**；存在多处并行/遗留实现（双创建路径、双奇遇系统）与一个死机制（stamina），需要一轮清理收敛；缺测试。

**改进优先级**：
1. 引入真正的服务端会话鉴权，所有 API 从 session 取 userId（堵 IDOR）。
2. 删除 `/api/cultivator` POST 与 `narrative` 的 ENCOUNTER 分支，收敛为单一实现。
3. 让 stamina 落地或移除。
4. 修复失败文案 bug，补充奇遇事件池（目前仅 3 个，重复度会很快显现）。
5. 补关键路径测试，把「每日」边界改为按用户时区或显式约定。
