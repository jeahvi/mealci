# MealCI · 麦门提交前流水线

> WorkBuddy × 麦当劳 1024 程序员节创意开发大赛参赛作品
> 把你的每一单麦当劳，当成一次待合入的 `git push`。

## 这是什么

MealCI 是一个 Agent Skill：你说"想吃点啥"，它把这一单当成一次代码提交，跑完 **lint → test → build → review** 四道流水线，最后给你一张 **MR（Merge Request）卡**——✅ Ready to merge 或 ⚠️ Changes requested。只有回复 `merge` 才会真正创建订单，支付永远由用户自己完成。

```
用户输入（想吃的 / 购物车 / "随便来点"）
        │
        ▼
  [Stage 1] lint  —— 营养检查，输出 warnings + diff 建议
        ▼
  [Stage 2] test  —— 优惠券组合穷举，输出测试报告
        ▼
  [Stage 3] build —— 门店取餐时长预估
        ▼
  [Stage 4] review —— 历史订单习惯评审，standup 风格评语
        ▼
   PR 卡（✅ Ready to merge / ⚠️ Changes requested）
        ▼
   用户确认 → 报价下单 → 支付由用户自己完成
```

## 为什么不是又一个热量计算器

麦当劳 MCP 把营养表、门店菜单、订单查询、优惠券、报价下单开放成了原子接口——单独拎出来都很"碎"。市面上大多数作品把它们当**查询接口**单点调用（热量计算、凑单、随机点餐），一次调用就结束。

MealCI 的差异在于**流程隐喻**：它不输出"答案"，输出"评审意见"。点餐是假的日常，CI 才是程序员真的日常——这正是 1024 程序员节该有的叙事。同时它把 6 个原子能力**编排成一条链**，而不是调用其中一个。

## 真实运行记录

以下输出全部来自 McDonald's MCP 真实数据（2026-10-10 实测，深圳）：

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Merge Request !1024   麦门主分支 ← feat/order
 门店: 麦当劳深圳同方信息港餐厅 (403m, 营业至 23:00)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 订单内容: 巨无霸 ×1 / 可乐 ×1

 lint:   ✅ 0 error, 1 warning
         └ ⚠ W401 sugary-drink: 可乐(中) 147 kcal / 36g 碳水
           → diff: 可乐 → 无糖可口可乐(中)  [-147 kcal]
 test:   ⚠ $COUPON 为空 — 该门店当前无可用券，按原价构建
 build:  ⏳ 到店自提 403m；今日可预约时段 夜市 17:14–21:45
 review: 💬 距上次 commit 已 265 天（2026-01-18，成都）。
            本次为深圳分部首个 commit，祝构建顺利。
──────────────────────────────
 应付金额: ¥35.0（巨无霸 ¥25.5 + 可乐 ¥9.5，来自报价接口真实数据）
 取餐方式: 堂食 / 外带取餐柜 可选
 状态: ✅ Ready to merge（已停在合入前，未调 create-order）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

其中"265 天"是用历史订单时间戳真实算出来的，不是写死的文案。

## 四道流水线

| Stage | 干什么 | 用到的 MCP 工具 |
|---|---|---|
| **lint** | 逐项检查营养（单品热量 ≥800 kcal 警告、订单总钠 ≥2400 mg 报错、含糖饮料检出），超线不拦截、只给 diff，回复 `apply 1` 一键回写购物车 | `list-nutrition-foods` |
| **test** | 枚举当前门店可用优惠券组合，输出"测试报告"，给出最优组合与节省金额；无券时输出 `$COUPON 为空` | `query-store-coupons` / `available-coupons` |
| **build** | 查附近门店与营业状态，估算取餐节奏 | `query-nearby-stores` |
| **review** | 拉历史订单做习惯评审："本周第 4 次麦辣鸡腿堡，这属于技术债，建议本单 refactor"；连续 5 单无蔬食报"检测到循环依赖" | `order-list` |
| **合入** | MR 卡展示真实报价，回复 `merge` 才创建订单 | `calculate-price` → `create-order` |

## 彩蛋

| 触发 | 输出 |
|---|---|
| 23:00–05:00 下单 | lint 切 strict 模式：`⚠ strict mode: 此单属于 hotfix，建议回滚（睡吧）` |
| 连续 5 单同款 | `检测到循环依赖，已尝试注入 break 语句（推荐一款没吃过的）` |
| merge 成功 | mock commit：`feat(order): 巨无霸×1 + 零度可乐×1, all checks passed 🍔` |
| 券为空 | `CI 环境变量缺失：$COUPON 为空，本单按原价构建` |

## 安装方法

1. 接入麦当劳官方 MCP：按 [MCP_INTEGRATION.md](./MCP_INTEGRATION.md) 在 WorkBuddy 中配置（配置示例见 [mcp-config.example.json](./mcp-config.example.json)，**Token 需自行到 https://open.mcd.cn/mcp 申请**）；
2. 启用连接器：WorkBuddy 左侧边栏【专家·技能·连接器】→【连接器】→ 右上角【自定义连接器】→ 配置 MCP → 粘贴配置并保存 → **启用 mcd-mcp**；
3. 导入 Skill：把本仓库的 `SKILL.md` 放入 WorkBuddy 技能目录（或通过技能导入入口加载）；
4. 验证：新开会话，输入「看看我的优惠券」确认 MCP 已连通，再输入「我想吃点啥」触发 MealCI。

**依赖**：WorkBuddy 客户端 + 麦当劳官方 MCP（Streamable HTTP，需自备 Token）。

## 使用示例

| 你说 | 它做什么 |
|---|---|
| `我想吃点啥` | 调菜单给 2-3 个候选 commit，选定后开跑四道流水线 |
| `看看这一单：巨无霸 + 可乐` | 完整跑 lint → test → build → review，输出 MR 卡 |
| `apply 1` | 采纳第 1 条 lint diff（如可乐换无糖），回写购物车并重跑 lint |
| `merge` | 确认下单，创建订单并返回支付链接（**支付由你本人完成**） |
| `close` | 放弃本次构建，不产生任何订单 |

完整真实输出见下方"真实运行记录"。

## 目标用户

- **点餐决策疲劳的上班族**：中午/深夜不知道吃什么，希望有依据、少纠结；
- **在意营养但不想被说教的人**：需要"可以商量的建议"（diff 式替换）而不是"禁止清单"；
- **有券但总忘记用的人**：券组合测试阶段会自动把可用券算进去；
- **麦当劳重度用户 / 有历史订单沉淀的人**：review 阶段基于真实历史做复盘，越用越准。

## 仓库结构

```
MealCI/
├── README.md               ← 项目介绍 / 安装方法 / 使用示例 / 目标用户
├── CONTEST_DECLARATION.md  ← 参赛声明（官方原件，内容未改动）
├── MCP_INTEGRATION.md      ← MCP Server / Tool / 调用流程 / 业务价值
├── mcp-config.example.json ← 脱敏 MCP 配置示例（仅环境变量占位符）
├── workbuddy.md            ← WorkBuddy 开发对话上下文（专项奖励凭证）
├── SKILL.md                ← Skill 本体（四道流水线规则、MR 卡模板、降级策略）
└── docs/
    ├── concept.md          ← 构思案（含被否决方案的留档）
    ├── smoke-test.md       ← 交付前自测清单（含实测结果）
    └── community-post.md   ← 社区发帖文案
```

## 合规与边界

- **支付永不代付**：Skill 只负责创建订单并给出支付提示，支付动作 100% 由用户完成；
- **数据不编造**：营养、价格、券、时长全部来自 MCP 真实返回；接口不可用时明确报 `runner offline`，不使用内置知识冒充查询结果；
- **不擅自下单**：任何订单都必须经过用户显式 `merge` 确认。

## 致谢

- McDonald's China MCP 开放平台（https://open.mcd.cn/mcp）
- WorkBuddy（腾讯）

---

*1024，一起把好点子变成真东西。*
