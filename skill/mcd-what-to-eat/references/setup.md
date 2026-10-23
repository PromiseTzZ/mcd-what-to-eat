# 麦当劳 MCP 接入 & 技能安装

本技能本身只是一个 Markdown 文件，不绑定客户端；但需要先连上麦当劳 MCP。

## 1. 申请 Token（每人一份）

1. 打开 <https://open.mcd.cn/mcp>
2. 手机号登录 → 右上角「控制台」→ 激活
3. 同意服务协议 → 复制 MCP Token

Token 绑定账号：券包、积分、订单查的都是这个账号的数据，**不要共用**。
Token 等同密码，不要提交到公开仓库、不要贴进群聊。服务免费，限流 600 请求/分钟。

## 2. 加 MCP 连接（所有客户端参数相同）

- 传输协议：**Streamable HTTP**
- URL：`https://mcp.mcd.cn`
- Header：`Authorization: Bearer <你的 Token>`

通用 JSON（多数客户端可直接粘贴导入）：

```json
{
  "mcpServers": {
    "mcd-mcp": {
      "type": "streamablehttp",
      "url": "https://mcp.mcd.cn",
      "headers": { "Authorization": "Bearer YOUR_MCP_TOKEN" }
    }
  }
}
```

> `type` 各客户端取值不同：Cherry Studio / Cursor 用 `streamablehttp`，**Qoder / Claude Code 用 `http`**，
> 部分老客户端只认 `sse`。报"传输类型无效 / Invalid transport"就换一个值试，`url` 和 `headers` 不变。

## 3. 各客户端安装位置

| 客户端 | 加 MCP | 装技能 |
|---|---|---|
| **Qoder** | 设置 → MCP → 添加：传输类型选 **HTTP**，地址 `https://mcp.mcd.cn`，Header `Authorization: Bearer <Token>`（JSON 里 `"type": "http"`，写 `streamablehttp` 会报"传输类型无效"） | 整个 `mcd-what-to-eat/` 放进 `~/.qoder/skills/` |
| **Claude Code** | `claude mcp add --transport http mcd-mcp https://mcp.mcd.cn --header "Authorization: Bearer <Token>"` | `~/.claude/skills/`（个人）或项目 `.claude/skills/` |
| **Codex CLI** | `~/.codex/config.toml` 加 `[mcp_servers.mcd]` 段 | 正文并入 `AGENTS.md` |
| **Gemini CLI** | `settings.json` 的 `mcpServers` 段 | 正文并入 `GEMINI.md` |
| **VSCode (Copilot)** | 工作区 `.vscode/mcp.json` | `.github/copilot-instructions.md` |
| **Cursor** | 设置 → Tools & MCP → Add Custom MCP，粘贴 JSON | 项目规则目录（以当前版本文档为准） |
| **TRAE** | 设置 → MCP → 手动添加，粘贴 JSON | 项目规则文件 |
| **Cherry Studio** | 设置 → MCP → 从 JSON 导入 | 无技能机制，把 SKILL.md 正文粘进助手提示词 |
| **DSH** | 用本包的 `mcp-bundle/`：把 `YOUR_MCP_TOKEN` 换成自己的，交给助手安装 | `~/.dsh/skills/` |

Codex CLI 的 TOML 大致长这样（字段名以当前版本文档为准）：

```toml
[mcp_servers.mcd]
url = "https://mcp.mcd.cn"
http_headers = { Authorization = "Bearer YOUR_MCP_TOKEN" }
```

> 路径以各客户端当前版本的文档为准。**任何客户端的通用兜底**：把 SKILL.md 正文粘进系统提示词或规则文件，工具名对上就能跑。

## 4. 验证

让助手调用 `now-time-info`，能返回服务器时间即接入成功。

## 5. 故障排查

- **401**：Token 错误或过期 → 回 open.mcd.cn 重新复制
- **连接失败，日志报 `Cannot convert argument to a ByteString` / 字符 20320**：请求头值里混进了中文，
  通常是占位符 `Bearer <你的 Token>` 原样粘贴没替换。换成真实 Token（纯英文数字）即可
- **429**：超过 600 次/分钟 → 等一分钟再试，别让客户端反复重连
- **工具名对不上**：本技能按后缀匹配（`query-meals`、`calculate-price` 等），前缀随客户端不同（`mcp__mcd-mcp__`、`mcp__mcd__` 或没有）
- **营养数据对不上菜单名**：正常。官方营养库和门店菜单是两套命名，技能里有三级溯源与折算规则
- **热量显示为 ≈区间或"未提供"**：官方无该品数据时的正常降级，不是故障
- **下单**：默认只推荐；明确说"下单"并确认取餐方式后才会调 create-order
