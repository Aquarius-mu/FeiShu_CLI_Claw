# lark-cli 使用规则（避坑指南）

处理任何飞书相关任务前，**必须先读取本文件**，再开始工作。

---

## 1. 命令参数速查

| 场景 | 正确写法 | 错误写法 |
|------|---------|---------|
| 搜索群组 | `im +chat-search --query "xxx" --as user` | 位置参数 ❌ |
| 搜索用户 | `contact +search-user --query "xxx" --as user` | `--keyword` ❌ |
| 获取群成员 | `im chat.members get --params '{"chat_id":"oc_xxx"}' --as user` | `--chat-id` flag ❌ |
| 获取 wiki 节点 | `wiki spaces get_node --params '{"token":"xxx"}' --as user` | `wiki nodes get` ❌ |
| 导出表格 | `sheets +export --output-path ./output.xlsx` | 绝对路径 `/tmp/` ❌ |
| path 参数 | 一律用 `--params '{"key":"val"}'` | 猜 flag 名 ❌ |

**@mention 语法**：
- 卡片 `lark_md` → `<at id="ou_xxx">姓名</at>`
- 文本消息 `text` → `<at user_id="ou_xxx">姓名</at>`
- `--text` 传 `@姓名` 是纯文本，无 @ 效果 ❌

---

## 2. 嵌入式 Bitable 访问流程

`sheets +info` 返回 `resource_type: "bitable"` 的页签时，不能直接用 spreadsheet token 作为 base_token，须走 3 步：

```bash
# Step 1：获取 blockToken
lark-cli api GET /open-apis/sheets/v2/spreadsheets/{spreadsheetToken}/metainfo --as user

# Step 2：解析 blockToken（格式：{base_token}_{table_id}）
# 例：BvXLbhpxPaybGNsfCTucV6VfnDb_tblshNr3dE1oWqU0
# → base_token = BvXLbhpxPaybGNsfCTucV6VfnDb，table_id = tblshNr3dE1oWqU0

# Step 3：读取数据
lark-cli base +record-list --base-token BvXLbhpxPaybGNsfCTucV6VfnDb \
  --table-id tblshNr3dE1oWqU0 --limit 100 --as user
```

---

## 3. Token 类型速查

| URL 来源 | Token 用途 | 处理方式 |
|---------|-----------|---------|
| `/wiki/{token}` | wiki node token | `wiki spaces get_node` 取 `obj_token` |
| `/sheets/{token}` | spreadsheet token | 直接用于 `sheets`；嵌入 bitable 须走 metainfo |
| `/base/{token}` | base_token | 直接用于 `base` 命令 |
| metainfo blockToken | `{base_token}_{table_id}` | `_` 分割，前半 base_token，后半（tbl 开头）table_id |

---

## 4. record-list 返回结构

```json
{
  "data": {
    "field_id_list": ["fldXxx", ...],   // 字段顺序
    "fields": ["活动名", "负责人", ...], // 字段名（同顺序）
    "data": [[val0, val1, ...], ...],   // 每行按 field_id_list 顺序
    "has_more": false
  }
}
```

`--field-id` 只能传单个字段，多字段时不传，本地按索引提取。

---

## 5. 执行效率原则

1. **先查缓存**：执行前先读 `~/feishu_bot/cache/`，已有的 chat_id、open_id、bitable token 无需重复查询。
   - **发版通知群 chat_id 固定**：`oc_36e42f5d801480d9ab554edbc230dcb9`（群名随版本更新，chat_id 不变，**无需搜索**）
2. **并行查询**：群/人员/文档搜索互相独立，一次性并行发出。
3. **schema 优先**：不确定参数时先 `lark-cli schema <resource>.<method>`，不要盲试。
4. **版本号映射**：`654.0` 对应表格页签 `654`（取整数部分）。
5. **查完更新缓存**：新解析到的 chat_id / bitable token 写入 `~/feishu_bot/cache/`。
6. **发版环境识别**：
   - `{version} 发好了` → 通用 | `9999发好了` → 9999服 | `测服发好了` → 测试服 | `正式服发好了` → 正式服

---

## 6. 卡片构建规则

### 6.1 禁止 `$(python3 -c "...")`

含特殊字符（引号、反引号）时 bash 会解析出错。**必须用 heredoc 或临时文件**：

```bash
# heredoc（推荐）
CARD=$(python3 - <<'PYEOF'
import json
card = { ... }
print(json.dumps(card, ensure_ascii=False))
PYEOF
)

# 临时文件
python3 /tmp/build_card.py > /tmp/card.json
CARD=$(cat /tmp/card.json)
```

### 6.2 icon.token 不可靠，一律用 emoji

`header.icon.token` 和 `div icon.token` 填写无效名称时静默不显示，且会导致标题内容消失。

**统一规则：所有图标直接用 Unicode emoji 写在 `lark_md` content 里，禁止依赖 `icon.token`。**

已验证可用的 token（仅供参考）：`warning-outlined`、`arrow-down-outlined`

### 6.3 schema 2.0 不支持 `note` tag

报错码 200861。替代方案：
```json
{"tag": "div", "text": {"tag": "lark_md", "content": "<font color=grey>备注文字</font>"}}
```

### 6.4 collapsible_panel 颜色枚举

用带层级后缀的颜色（`blue-100`、`grey` 等），**不支持无后缀基础色名**（`"blue"` 单用报错 11310）。

### 6.5 lark_md Emoji Key 规则

**严格规定**：只能使用[官方列表](https://open.feishu.cn/document/server-docs/im-v1/message-reaction/emojis-introduce)中的 key，禁用未收录名称（`:WAITING:`、`:PROCESSING:` 等渲染成纯文本）。key 大小写必须原样（`Trophy` 不是 `TROPHY`）。

**`<text_tag>` 内不渲染 `:KEY:`**，须改用 Unicode emoji（如 `✅`）。

#### 发版通知常用
```
:Loudspeaker:  :OnIt:   :DONE:   :THUMBSUP:   :APPLAUSE:
:PARTY:        :Trophy: :Fire:   :THANKS:     :CheckMark:
```

#### 完整官方 key（按大小写原样使用）
```
OK, THUMBSUP, THANKS, MUSCLE, FINGERHEART, APPLAUSE, FISTBUMP, JIAYI, DONE,
SMILE, BLUSH, LAUGH, SMIRK, LOL, FACEPALM, LOVE, WINK, PROUD, WITTY, SMART,
SCOWL, THINKING, SOB, CRY, ERROR, NOSEPICK, HAUGHTY, SLAP, SPITBLOOD, TOASTED,
GLANCE, DULL, INNOCENTSMILE, JOYFUL, WOW, TRICK, YEAH, ENOUGH, TEARS,
EMBARRASSED, KISS, SMOOCH, DROOL, OBSESSED, MONEY, TEASE, SHOWOFF, COMFORT,
CLAP, PRAISE, STRIVE, XBLUSH, SILENT, WAVE, WHAT, FROWN, SHY, DIZZY, LOOKDOWN,
CHUCKLE, WAIL, CRAZY, WHIMPER, HUG, BLUBBER, WRONGED, HUSKY, SHHH, SMUG,
ANGRY, HAMMER, SHOCKED, TERROR, PETRIFIED, SKULL, SWEAT, SPEECHLESS, SLEEP,
DROWSY, YAWN, SICK, PUKE, BETRAYED, HEADSET,
EatingFood, MeMeMe, Sigh, Typing, Lemon, Get, LGTM, OnIt, OneSecond,
VRHeadset, YouAreTheBest, SALUTE, SHAKE, HIGHFIVE, UPPERLEFT, ThumbsDown,
SLIGHT, TONGUE, EYESCLOSED, RoarForYou, CALF, BEAR, BULL, RAINBOWPUKE,
ROSE, HEART, PARTY, LIPS, BEER, CAKE, GIFT, CUCUMBER, Drumstick, Pepper,
CANDIEDHAWS, BubbleTea, Coffee, Yes, No, OKR, CheckMark, CrossMark,
MinusOne, Hundred, AWESOMEN, Pin, Alarm, Loudspeaker, Trophy, Fire, BOMB,
Music, XmasTree, Snowman, XmasHat, FIREWORKS, 2022, REDPACKET, FORTUNE,
LUCK, FIRECRACKER, StickyRiceBalls, HEARTBROKEN, POOP, StatusFlashOfInspiration,
18X, CLEAVER, Soccer, Basketball, GeneralDoNotDisturb, Status_PrivateMessage,
GeneralInMeetingBusy, StatusReading, StatusInFlight, GeneralBusinessTrip,
GeneralWorkFromHome, StatusEnjoyLife, GeneralTravellingCar, StatusBus,
GeneralSun, GeneralMoonRest, MoonRabbit, Mooncake, JubilantRabbit, TV,
Movie, Pumpkin, BeamingFace, Delighted, ColdSweat, FullMoonFace, Partying,
GoGoGo, ThanksFace, SaluteFace, Shrug, ClownFace, HappyDragon
```
