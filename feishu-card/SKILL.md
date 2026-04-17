---
name: feishu-card
version: 2.0.0
description: "飞书交互卡片（schema 2.0）：构建和发送卡片消息，含 @提及、流式思考态、原地更新、按钮/选择器交互。通过 lark-cli --as bot 发送，无需手动管理 token。"
metadata:
  requires:
    bins: ["lark-cli", "jq", "python3"]
---

# feishu-card（schema 2.0）

## 关键规则

- 所有卡片使用 **schema 2.0**，元素放在 `body.elements[]`，不是顶层 `elements[]`
- 发送卡片必须 `--as bot`，`--msg-type interactive`，`--content` 传 JSON 字符串
- 卡片内 @人 用 `<at id="ou_xxx">姓名</at>`（不是 `<at user_id>`）
- 先发"思考中"卡片拿到 `message_id`，Claude 回答后 PATCH 原地更新，避免刷屏

---

## 卡片基础结构

```json
{
  "schema": "2.0",
  "config": {"streaming_mode": false},
  "header": {
    "title": {"tag": "plain_text", "content": "标题"},
    "template": "blue"
  },
  "body": {
    "elements": [
      {"tag": "markdown", "content": "正文内容，支持 **加粗** _斜体_ `代码`"},
      {"tag": "hr"},
      {"tag": "markdown", "content": "第二段"}
    ]
  }
}
```

**header template 颜色**：`blue` / `green` / `red` / `grey` / `orange` / `purple` / `wathet`

---

## 常用元素

| tag | 说明 | 示例 |
|-----|------|------|
| `markdown` | Markdown 文本（支持 @、加粗、链接、表格） | `{"tag":"markdown","content":"**文本**"}` |
| `hr` | 分割线 | `{"tag":"hr"}` |
| `action` | 按钮组 | 见下方 |
| `note` | 底部备注（小字） | `{"tag":"note","elements":[{"tag":"plain_text","content":"备注"}]}` |
| `column_set` | 多列布局 | `{"tag":"column_set","flex_mode":"none","columns":[...]}` |

---

## @提及语法

| 消息类型 | @人语法 | @全体 |
|---------|---------|-------|
| 卡片 `interactive` lark_md | `<at id="ou_xxx">姓名</at>` | `<at id="all">所有人</at>` |
| 文本 `text` / `post` | `<at user_id="ou_xxx">姓名</at>` | `<at user_id="all"></at>` |

> `+messages-send` / `+messages-reply` 快捷方式会把 `<at id=...>` 自动归一化成 `user_id`，但卡片内 `lark_md` 只认 `id=`，不要混用。

---

## 发送 / 回复 / 更新

### 发送到群（新消息）
```bash
CARD=$(python3 -c "
import json
card = {
  'schema': '2.0',
  'header': {'title': {'tag': 'plain_text', 'content': '标题'}, 'template': 'blue'},
  'body': {'elements': [{'tag': 'markdown', 'content': '内容'}]}
}
print(json.dumps(card))
")
lark-cli im +messages-send \
  --chat-id oc_xxx \
  --msg-type interactive \
  --content "$CARD" \
  --as bot
```

### 回复指定消息（获取 message_id 供后续更新）
```bash
MSG_ID=$(lark-cli im +messages-reply \
  --message-id om_xxx \
  --msg-type interactive \
  --content "$CARD" \
  --as bot \
  --jq '.data.message_id')
```

### 回复到 thread（不出现在主聊天流）
```bash
lark-cli im +messages-reply \
  --message-id om_xxx \
  --msg-type interactive \
  --content "$CARD" \
  --reply-in-thread \
  --as bot
```

### 防重复发送（idempotency-key，1小时内相同 key 只发一次）
```bash
lark-cli im +messages-send \
  --chat-id oc_xxx \
  --msg-type interactive \
  --content "$CARD" \
  --idempotency-key "deploy-654.0-notify" \
  --as bot
```

### 原地更新卡片（PATCH）
```bash
PAYLOAD=$(jq -n --argjson card "$CARD" '{"msg_type":"interactive","content":($card|tojson)}')
lark-cli api PATCH "/open-apis/im/v1/messages/${MSG_ID}" \
  --data "$PAYLOAD" \
  --as bot
```

---

## 卡片模板

### 思考中（流式态，先发占位）
```bash
make_thinking_card() {
  local name="$1" question="$2"
  jq -n --arg name "$name" --arg q "$question" '{
    schema:"2.0",
    header:{title:{tag:"plain_text",content:("💬 "+$name+" 问")},template:"grey"},
    config:{streaming_mode:true},
    body:{elements:[
      {tag:"markdown",content:("> "+$q)},
      {tag:"hr"},
      {tag:"markdown",content:"⏳ **思考中…**"}
    ]}
  }'
}
```

### 回答完成（更新思考中卡片）
```bash
make_reply_card() {
  local name="$1" question="$2" answer="$3" duration="$4"
  jq -n --arg name "$name" --arg q "$question" --arg ans "$answer" --arg dur "$duration" '{
    schema:"2.0",
    header:{title:{tag:"plain_text",content:("💬 "+$name+" 问")},template:"blue"},
    config:{streaming_mode:false},
    body:{elements:[
      {tag:"markdown",content:("> "+$q)},
      {tag:"hr"},
      {tag:"markdown",content:$ans},
      {tag:"hr"},
      {tag:"markdown",content:("✅ 耗时 **"+$dur+"**")}
    ]}
  }'
}
```

### 错误提示
```bash
make_error_card() {
  local msg="${1:-抱歉，我暂时无法回答，请稍后再试。}" hint="${2:-}"
  local content="$msg"
  [[ -n "$hint" ]] && content+="\\n\\n💡 _${hint}_"
  jq -n --arg content "$content" '{
    schema:"2.0",
    header:{title:{tag:"plain_text",content:"❌ 出错了"},template:"red"},
    config:{streaming_mode:false},
    body:{elements:[{tag:"markdown",content:$content}]}
  }'
}
```

### 成功通知
```bash
make_success_card() {
  jq -n --arg title "$1" --arg content "$2" '{
    schema:"2.0",
    header:{title:{tag:"plain_text",content:$title},template:"green"},
    config:{streaming_mode:false},
    body:{elements:[{tag:"markdown",content:$content}]}
  }'
}
```

### 权限不足
```bash
make_noperm_card() {
  jq -n '{
    schema:"2.0",
    header:{title:{tag:"plain_text",content:"⛔ 权限不足"},template:"red"},
    config:{streaming_mode:false},
    body:{elements:[{tag:"markdown",content:"抱歉，你没有使用该 Bot 的权限。"}]}
  }'
}
```

---

## 按钮交互

```bash
jq -n '{
  schema:"2.0",
  header:{title:{tag:"plain_text",content:"请确认"},template:"orange"},
  body:{elements:[
    {tag:"markdown",content:"是否执行此操作？"},
    {tag:"action",actions:[
      {tag:"button",text:{tag:"plain_text",content:"确认"},type:"primary",
       value:{action:"confirm",data:"payload"}},
      {tag:"button",text:{tag:"plain_text",content:"取消"},type:"default",
       value:{action:"cancel"}}
    ]}
  ]}
}'
```

按钮类型：`primary`（蓝）/ `default`（灰）/ `danger`（红）

用户点击后飞书推送 `card.action.trigger` 事件，`action.value` 即按钮的 `value` 字段。

---

## 发版通知（含 @人）模板

```bash
send_deploy_notify() {
  local chat_id="$1" version="$2" env="$3"
  shift 3
  local at_list=""
  for open_id in "$@"; do
    at_list+="<at id=\"${open_id}\">${open_id}</at> "
  done
  CARD=$(python3 -c "
import json, sys
card = {
  'schema': '2.0',
  'header': {'title': {'tag': 'plain_text', 'content': '【${version} ${env}】已发好 🎉'}, 'template': 'green'},
  'body': {'elements': [
    {'tag': 'markdown', 'content': '<at id=\"ou_atlas\">xxx(Atlas)</at> <at id=\"ou_scott\">xx(Scott)</at>\n请知悉～'},
    {'tag': 'hr'},
    {'tag': 'markdown', 'content': '${at_list}\n麻烦配置活动，辛苦了！'}
  ]}
}
print(json.dumps(card))
")
  lark-cli im +messages-send \
    --chat-id "$chat_id" \
    --msg-type interactive \
    --content "$CARD" \
    --as bot
}
```

---

## 错误处理

```bash
result=$(lark-cli api PATCH "/open-apis/im/v1/messages/${MSG_ID}" \
  --data "$PAYLOAD" --as bot 2>/dev/null)
code=$(echo "$result" | jq -r '.code // 0')
if [[ "$code" != "0" ]]; then
  # PATCH 失败（消息过期/权限问题），降级为发新消息
  lark-cli im +messages-send --chat-id "$CHAT_ID" --msg-type interactive \
    --content "$CARD" --as bot
fi
```

常见错误码：
| code | 含义 | 处理 |
|------|------|------|
| 230002 | 没有操作权限 | 检查 bot 是否在群内 |
| 10012 | 消息不存在或已过期 | 降级发新消息 |
| 99991663 | token 过期 | 重新 `lark-cli auth login --as bot` |
