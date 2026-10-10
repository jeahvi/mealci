# MCP 接入说明

MealCI 依赖**麦当劳中国官方 MCP**。本文说明如何获取 Token 并在 WorkBuddy 中完成接入，以及实测确认的接口细节。

## 一、获取 MCP Token

1. 访问麦当劳开放平台：**https://open.mcd.cn/mcp**
2. 右上角【登录】，使用手机号验证登录；
3. 登录后点右上角「控制台」→ 激活 → 同意服务协议；
4. Token 生成后一键复制，妥善保存（**不要提交到公开仓库**）。

## 二、接入参数（官方）

| 项 | 值 |
|---|---|
| 服务器接入地址 | `https://mcp.mcd.cn/mcp-servers/mcd-mcp` |
| 传输协议 | Streamable HTTP（不支持 WebSocket） |
| MCP 协议版本 | 2025-06-18 及之前（实测 2025-06-18 可用） |
| 认证方式 | 请求头 `Authorization: Bearer <YOUR_MCP_TOKEN>` |
| 限流 | 每个 Token 每分钟最多 600 次请求，超出返回 429 |
| 适用地区 | 中国大陆（不含港澳台） |

错误码：

| code | 原因 | 处理 |
|---|---|---|
| 401 | Token 无效 / 过期 / 未提供 | 检查 Authorization 头 |
| 429 | 触发限流 | 降低请求频率 |

## 三、在 WorkBuddy 中配置

编辑 `~/.workbuddy/mcp.json`，在 `mcpServers` 中加入：

```json
{
  "mcpServers": {
    "mcd-mcp": {
      "url": "https://mcp.mcd.cn/mcp-servers/mcd-mcp",
      "headers": {
        "Authorization": "Bearer YOUR_MCP_TOKEN"
      },
      "disabled": false
    }
  }
}
```

保存后打开 WorkBuddy 的**连接器管理页 → 自定义连接器**，对 `mcd-mcp` 点「信任」启用。启用后可用 `tools/list` 确认连接正常。

## 四、工具清单

官方文档列出 24 个工具，**实测 `tools/list` 实际返回 35 个**（以运行时为准）。MealCI 用到的核心工具：

| 用途 | 工具名 | 说明 |
|---|---|---|
| lint 数据源 | `list-nutrition-foods` | 餐品营养列表（能量/蛋白质/脂肪/碳水/钠/钙） |
| 菜单与商品编码 | `query-meals` | 查门店可售餐品，拿 `productCode` |
| 餐品详情 | `query-meal-detail` | 套餐组成与可替换项 |
| test 数据源 | `query-store-coupons` | 指定门店+订单类型下可用券 |
| 券包 | `query-my-coupons` / `available-coupons` / `auto-bind-coupons` | 我的券 / 可领券 / 一键领券 |
| build 数据源 | `query-nearby-stores` | 到店场景查门店（`beType=1` 自提 / `5` 得来速） |
| 外送门店 | `delivery-query-stores` / `delivery-query-addresses` | 麦乐送场景 |
| review 数据源 | `order-list` / `query-order` | 历史订单 / 订单详情 |
| 报价 | `calculate-price` | 计算金额（单位：**分**） |
| 下单 | `create-order` / `cancel-order` | 创建 / 取消订单，返回支付链接 |
| 时间 | `now-time-info` | 服务器时间（彩蛋判断用） |
| 其他 | `campaign-calendar`、奖励/积分类 `query-my-account`、`mall-*` 等 | 活动日历、积分商城 |

## 五、实测踩坑（重要）

1. **工具数不等于文档数**：官方文档页只列 24 个，实际 35 个。比如文档说"没有提供查看订单记录"，但运行时存在 `order-list` 和 `query-order`——MealCI 的 review 阶段正是靠它成立的。**以 `tools/list` 为准**。
2. **营养接口返回 toon 紧凑格式**，不是标准 JSON 数组：
   ```
   {productName,nutritionDescription,energyKj,energyKcal,protein,fat,carbohydrate,sodium,calcium}:
     猪柳麦满分,null,1288,308,16,16,24,781,213
   ```
   第一行是表头，需按逗号切列解析。
3. **金额单位是"分"**：`calculate-price` 返回 `3500` 表示 ¥35.00，展示前必须除以 100。
4. **`storeCode` 必须全数字**（如 `1420762`），`beType`/`orderType` 的传参组合必须严格按场景：到店自取 `orderType=1` 且不传 `beCode`；得来速 `orderType=1` 且必传 `beCode`；外送 `orderType=2` 且必传 `beCode`。
5. **限流 600 次/分钟**：券组合穷举时注意控制频率。
6. 门店 `query-nearby-stores` 的 `searchType=2` 需同时传 `city` 与 `keyword`；`reservation=true` 时要完整展示 `reservationTimeOptions`，否则引导用户预约会失败。

## 六、安全提醒

- Token 属于个人凭证，**请勿写入公开仓库或截图外传**；本仓库不包含任何真实 Token；
- MealCI 不会代付、不索取支付凭据，`create-order` 返回的支付链接需用户自行完成支付。
