# 🤖 Feishu_bot_CLI · Claude Code GamePlay Agent

> 部署在开发机上的 Claude Code 飞书对话助手，专为游戏项目策划 / QA / 运营设计。

![Platform](https://img.shields.io/badge/platform-Linux-blue)
![Shell](https://img.shields.io/badge/shell-bash-green)
![Claude](https://img.shields.io/badge/powered%20by-Claude%20Code-orange)
![lark-cli](https://img.shields.io/badge/lark--cli-%E2%89%A51.0.7-brightgreen)

---

## ✨ 能做什么

| | 能力 | 描述 |
|--|------|------|
| 🗂️ | **案子协作** | @ Bot 审阅需求、整理思路、@相关人员跟进 |
| 🎮 | **游戏后台运维** | 自然语言发 GM 指令、批量操作区服、跨区拷贝角色 |
| 📨 | **飞书全场景操作** | 发消息、读文档/多维表格、搜用户，全程 lark-cli |
| 🃏 | **富卡片回复** | schema 2.0 全组件（表格、折叠面板、多列布局、@提及） |

---

## 🏗️ 架构

```
飞书群消息（@ Bot）
  │
  ├─ lark-cli event +subscribe     # 事件监听
  │
  ├─ lark_sweet_bot.sh             # 会话管理 / 并发优化 / 卡片构建
  │    ├─ 预建 Session 池  ──── 消除冷启动延迟
  │    ├─ 群隔离 Session   ──── 每个群独立上下文
  │    ├─ 思考态占位卡片   ──── 先发占位，完成后原地更新
  │    ├─ CARD_JSON:: 协议 ──── Claude 直出卡片 JSON，跳过文本转换
  │    └─ 超长自动折叠     ──── >1500 字自动 collapsible_panel
  │
  ├─ Claude Code CLI --resume      # 带上下文持续调用
  │    └─ Skills × 4  ──── feishu-lark-cli / feishu-card / feishu-cache / feishu-file-ops
  │
  └─ lark-cli im +messages-send    # 卡片回复
```

---

## 🔌 Skills

| Skill | 职责 |
|-------|------|
| `feishu-lark-cli` | lark-cli 完整操作指南 + 所有避坑规则 |
| `feishu-card` | schema 2.0 卡片构建、流式态、原地更新、@提及 |
| `feishu-cache` | 缓存读写，并发安全，TTL 管理 |
| `feishu-file-ops` | 文件操作白名单，防误写敏感路径 |

> 对话结束后异步触发 `trigger_skill_improve`，Claude 自主优化 Skill 文件。

---

## 🎮 GM Server — 游戏后台 MCP

内置 MCP 服务，Claude 直接通过自然语言操作游戏后台，无需登录 GM 页面。

| 工具 | 说明 |
|------|------|
| `list_zones` | 查询区服列表（5 分钟缓存，支持关键词过滤） |
| `search_gm_commands` | 搜索 GM 指令，支持中英文，最多返回 20 条 |
| `send_gm_command` | 向指定区服发送单条 GM 指令 |
| `batch_send_gm_command` | 并发批量发送，最多 100 区，并发上限 20 |
| `copy_role` | 跨区拷贝角色（支持 dev / and / ali / cnf / cnt） |

**配置** `~/.claude/mcp-servers/gm_server/config.json`：

```json
{
  "base_url": "http://your-gm-backend/",
  "token": "your_token",
  "nick": "your_nick",
  "gm_cmd_js_path": "/path/to/gmweb/webback/js/gm_cmd.js"
}
```

> `token` 与 `username/password` 二选一；配置 `gm_cmd_js_path` 后支持中英文指令搜索。

---

## 🃏 飞书卡片速查（schema 2.0）

**支持组件**

`div` · `markdown` · `img` · `table` · `hr` · `column_set` · `collapsible_panel` · `button` · `input` · `select_static`

**@提及语法**

```
卡片 lark_md  →  <at id="ou_xxx">姓名</at>
文本消息      →  <at user_id="ou_xxx">姓名</at>
```

**颜色枚举**：格式 `{色}-{层级}`，如 `blue-100`、`green-200`、`orange-50`、`grey`
> ⚠️ 禁用无后缀色名（`"blue"` 报错 11310）；不支持 RGBA

**Emoji**：`:DONE:` `:OK:` `:WAITING:` `:ERROR:` `:FIRE:` `:ROCKET:` `:THUMBSUP:` `:GIFT:` …
Unicode 🎉 ✅ ❌ ⚠️ 🚀 可直接写在标题和 lark_md 中

> ⚠️ `note` tag 在 schema 2.0 已移除（报错 200861）
> 替代方案：`{"tag":"div","text":{"tag":"lark_md","content":"<font color=grey>备注</font>"}}`

---

## ⚡ lark-cli 常用命令

```bash
# 搜群 / 搜用户
lark-cli im +chat-search --query "群名" --as user
lark-cli contact +search-user --query "姓名" --as user

# 获取群成员
lark-cli im chat.members get --params '{"chat_id":"oc_xxx"}' --as user --page-all

# 读知识库节点
lark-cli wiki spaces get_node --params '{"token":"wiki_token"}' --as user

# 导出表格（只能用相对路径）
lark-cli sheets +export --spreadsheet-token xxx --file-extension xlsx --output-path ./out.xlsx
```

> ⚠️ 构建卡片 JSON **必须用 heredoc**，`$(python3 -c "...")` 遇反引号会崩：

```bash
CARD=$(python3 - <<'PYEOF'
import json
card = {"schema":"2.0","header":{"title":{"tag":"plain_text","content":"标题"},"template":"blue"},"body":{"elements":[]}}
print(json.dumps(card))
PYEOF
)
lark-cli im +messages-send --chat-id oc_xxx --msg-type interactive --content "$CARD" --as bot
```

---

## 🚀 安装

```bash
# 1. 克隆仓库
git clone <repo_url> && cd <repo>

# 2. 安装 lark-cli
npm install -g @larksuiteoapi/lark-cli

# 3. 两个身份都要登录
lark-cli auth login --as user
lark-cli auth login --as bot

# 4. 安装 Claude Code CLI → https://github.com/anthropics/claude-code

# 5. 安装 Skills
cp -r skills/feishu-* ~/.claude/skills/
```

**编辑 `lark_sweet_bot.sh` 顶部变量：**

```bash
LARK=/path/to/lark-cli
BOT_OPEN_ID=ou_xxx        # 机器人 open_id（飞书开放平台查看）
ALLOWED_SENDER=ou_xxx     # 允许使用的用户 open_id
MCP_CONFIG=/path/to/.claude/mcp.json
```

**飞书应用权限**（开放平台配置）：

```
im:message  ·  im:chat  ·  contact:user.base:readonly
wiki:wiki:readonly（可选）  ·  sheets:spreadsheet:readonly（可选）  ·  bitable:app:readonly（可选）
订阅事件：im.message.receive_v1
```

---

## 🖥️ 启动与管理

```bash
./lark_sweet_bot.sh start      # 后台启动
./lark_sweet_bot.sh status     # 查看状态
./lark_sweet_bot.sh stop       # 停止
./lark_sweet_bot.sh restart    # 重启
tail -f ~/feishu_bot/bot.log   # 实时日志
```

**群内指令**：`@ Bot + 消息` 对话 · `/clear` 清除上下文 · `/help` 帮助

---

## ⚠️ 注意事项

- Bot 使用 `--dangerously-skip-permissions` 启动 Claude，**请勿在公网直接暴露**
- `ALLOWED_SENDER` 白名单外的用户收到「权限不足」提示卡片
- Skill 自动改进异步执行，不影响响应速度，日志写入 `~/feishu_bot/skill_improve.log`
- `MCP_CONFIG` 须包含 `gm-server` 配置，Claude 才能调用 GM 工具
