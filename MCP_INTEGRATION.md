# MCP 集成说明

本项目全部决策数据来自麦当劳官方 MCP Server，不做任何本地硬编码菜单、价格或热量。

## 使用的 MCP Server

| 项目 | 说明 |
|---|---|
| Server 名称 | mcd-mcp |
| 接入地址 | `https://mcp.mcd.cn` |
| 传输协议 | Streamable HTTP |
| 认证方式 | Bearer Token（MCP Token） |
| 速率限制 | 600 次/分钟/Token |
| 官方文档 | https://github.com/M-China/mcd-mcp-server |
| 开放平台 | https://open.mcd.cn/mcp |

Token 跟随注册账号，券包、积分、订单查的都是该账号的数据，因此每位使用者需自行申请，不共用。

## 配置方式

```json
{
  "mcpServers": {
    "mcd-mcp": {
      "type": "streamablehttp",
      "url": "https://mcp.mcd.cn",
      "headers": {
        "Authorization": "Bearer ${MCD_MCP_TOKEN}"
      }
    }
  }
}
```

Token 通过环境变量 `MCD_MCP_TOKEN` 注入，不写入仓库。

`type` 字段各客户端取值不同：Cherry Studio / Cursor / WorkBuddy 用 `streamablehttp`，Qoder / Claude Code 用 `http`，部分老客户端只认 `sse`。`url` 与 `headers` 不变。

## 使用的 MCP Tools

本项目使用 13 个麦当劳 MCP 工具，按工作流阶段分组。

### 时段与门店（Step 1）

| Tool Name | 功能 | 调用时机 |
|---|---|---|
| `now-time-info` | 获取服务器当前时间 | 判断早餐/午餐/下午茶时段，计算券的剩余有效期 |
| `query-nearby-stores` | 按城市 + 关键词搜附近门店 | 到店取餐场景（默认） |
| `delivery-query-addresses` | 查询用户配送地址 | 外送场景 |
| `delivery-query-stores` | 查询地址附近可配送门店 | 外送场景 |

**业务价值**：时段决定菜单范围（早餐和正餐是两套菜单），门店决定 `storeCode`——所有后续价格与库存都以门店为维度，选错门店等于全部数据作废。

### 菜单（Step 2）

| Tool Name | 功能 | 调用时机 |
|---|---|---|
| `query-meals` | 拉门店当前可售餐品列表 | 每次推荐必调，作为候选池 |
| `query-meal-detail` | 查套餐组成、可选替换、默认选项 | 只对候选前几名调用，避免全量拉取 |

**业务价值**：`query-meals` 返回的是按分类组织的餐品编码与标签，不含组成；套餐的真实热量和价格必须靠 `query-meal-detail` 拆到子品才能算准。按需调用而非全量调用，把一次推荐的接口消耗压在可控范围内。

### 营养（Step 3）

| Tool Name | 功能 | 调用时机 |
|---|---|---|
| `list-nutrition-foods` | 官方营养库，一次返回约 160 条单品数据 | 需要热量时调用一次，本地按名称匹配 |

**业务价值**：这是本项目与其他同类 Skill 的核心差异点。营养库只收录单品、不含套餐，且与门店菜单是两套命名体系，需要按名称做关键词匹配、套餐按子品逐项相加。数据格式为表头 `[160]{productName,nutritionDescription,energyKj,energyKcal,...}` 加逐行数值，**必须取第 4 列 `energyKcal`**，第 3 列是千焦，误用会导致 4.18 倍误差。

### 优惠券（Step 4）

| Tool Name | 功能 | 调用时机 |
|---|---|---|
| `available-coupons` | 查询当前可领取的麦麦省券 | 获取平台层可领券 |
| `query-store-coupons` | 查询当前门店可用券 | 获取门店层券，含有效期与门槛 |
| `query-my-coupons` | 查询用户已持有的券 | 用户已登录时补充 |

**业务价值**：券的可用性由「有效期 + 门槛 + 限定品类 + 限定时段」四个条件共同决定。本项目在算价前先校验这四项，**过期券不会被算进「券后价」**——这是同类工具最常见的错误来源（展示一个明天根本用不了的券后价）。

### 算价与下单（Step 5）

| Tool Name | 功能 | 调用时机 |
|---|---|---|
| `calculate-price` | 按商品列表 + 券计算应付金额 | 每个候选组合都实算，不心算 |
| `create-order` | 创建订单 | **仅在用户明确确认后** |
| `cancel-order` | 取消订单 | **仅在用户明确要求后** |

**业务价值**：`calculate-price` 是唯一可信的价格来源，返回的金额单位是**分**，需除以 100。本项目禁止模型自行推算券后价，所有表格中的价格都来自该接口的实际返回。下单类工具默认不触发，必须用户显式确认。

## 调用流程

```
now-time-info
    │  判定时段
    ▼
query-nearby-stores (searchType=2, city + keyword)
    │  拿到 storeCode
    ▼
query-meals ──────────────► 按分类建候选池 + 按时段过滤
    │
    ├──► query-meal-detail （只对候选前几名）
    │        拆套餐组成 → 子品列表
    ▼
list-nutrition-foods （一次，本地匹配）
    │  A 级精确值 → 未命中走 B/C 级
    ▼
available-coupons + query-store-coupons
    │  校验有效期 / 门槛 / 品类 / 时段
    ▼
calculate-price （循环，每个候选一次）
    │
    ▼
五档推荐：最低卡 / 最省钱 / 兼顾 / 惊喜 / 自由搭
    │
    ▼
"要下单吗？" ──用户明确确认──► create-order
```

## 实测调用契约

以下为使过程中验证过的参数约束，写在这里便于二次开发：

- 门店类工具（`query-meals` / `query-meal-detail` / `calculate-price` / `query-store-coupons`）三个必填：`storeCode`、`orderType`、`beType`。
  `beType` 取值：1 到店取餐 / 2 麦乐送 / 5 得来速 / 6 团餐。
- **到店自取（`orderType=1` 且 `beType=1`）不要传 `beCode`**，传了会报错；只有外送、得来速、团餐才需要。
- `query-nearby-stores` 没有经纬度参数，只有 `city` + `keyword`，靠 `searchType=2` 按位置搜索；`searchType=1` 是搜收藏门店（收藏为空会返回 `600050`）。
- `reservationDate`（`yyyy-MM-dd HH:mm`）只在预约场景传，非预约不传；跨天推荐（如"明天午餐"）必须传，否则取到的是今天的档期。
- 工具名带客户端前缀，`mcp__mcd-mcp__query-meals`（Qoder / Claude Code）、`mcp__mcd__query-meals`（DSH）或没有前缀。**按后缀匹配，不要按前缀判断是否已接入。**

## 错误处理

| 错误 | 原因 | 处理方式 |
|---|---|---|
| 401 | Token 无效、过期或未提供 | 提示用户去 open.mcd.cn 重新复制，不重试 |
| 429 | 超过 600 次/分钟 | 建议稍后再试，不连续重试 |
| `600050` | 收藏门店列表为空 | 改用 `searchType=2` 按位置搜索 |
| 券后价高于原价 | 券门槛未满足 | 换下一张券重新 `calculate-price` |

营养数据缺失不属于错误，按 SKILL.md 中的 A→B→C→D 四级溯源降级处理，并在输出中标注来源。
