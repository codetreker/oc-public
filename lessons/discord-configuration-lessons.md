# Discord 配置经验总结(OpenClaw 多 Agent)

最后更新:2026-04-01

## 适用场景

- OpenClaw 多账号(main / architect / pm / dev / qa)
- Discord 群聊协作
- 需要 agent 间互相点名触发

## 一、核心结论(先看这个)

> 团队通讯录见:[references/discord-team-directory.md](../references/discord-team-directory.md)

1. **必须使用真实 mention**:`<@USER_ID>`
   - 纯文本 `@名字` 在很多情况下不会出现在 `mentions[]`,触发失败。
2. **bot 间沟通要显式放行**:每个账号配置 `allowBots: "mentions"`
   - 否则 bot 发给 bot 的消息可能被忽略。
3. **群聊触发建议开启 `requireMention: true`**
   - 能有效防止刷屏和误触发。
4. **Message Content Intent 必须正确配置**
   - 缺失时可能出现 gateway `4014`,导致收消息异常。
5. **streaming 建议统一关闭(`"off"`)**
   - 对协作链路验收帮助不大,容易增加排障噪音。

---

## 二、推荐配置要点(实战版)

### 1) 账号绑定

- 通过 `bindings[]` 显式把 Discord account 绑定到对应 agent。
- 建议每个角色一个独立 bot 账号,职责清晰,审计方便。

### 2) 群策略

- `groupPolicy: "open"`
- `guilds: { "*": { "requireMention": true } }`

这样可以保证:
- 被点名才处理
- 普通闲聊不抢答

### 3) bot-to-bot 放行

建议每个 Discord 账号都配置:

```json
"allowBots": "mentions"
```

否则典型现象:
- 人类 @ bot 可以触发
- bot @ bot 不触发

### 4) streaming

统一设置:

```json
"streaming": "off"
```

注意保持顶层和账号级一致,避免行为不一致。

---

## 三、典型故障与排查路径

### 故障 A:@了没反应

优先检查:
1. 消息里是否是 `<@ID>` 真实 mention(看 `mentions[]` 是否非空)
2. 目标 bot 是否在线且 token 正确
3. 是否被 `requireMention` / 群策略过滤

### 故障 B:bot 间互相 @ 不触发

优先检查:
1. 两边账号是否都配置了 `allowBots: "mentions"`
2. 是否真实 mention 到正确 ID
3. 是否命中了预处理日志 `reason: no-mention`

### 故障 C:连接不稳定 / 收不到内容

优先检查:
1. Discord Developer Portal 是否开启 Message Content Intent
2. 日志是否有 gateway `4014`
3. 网关是否 hot reload 成功

---

## 四、验收方法(建议固定流程)

1. 由 A bot 发送一条同时真实 mention B/C/D 的消息。
2. 要求 B/C/D 分别回固定短码(例如 `B-OK`、`C-OK`、`D-OK`)。
3. 用消息读取接口确认:
   - 回复内容
   - `author.id` 对应正确 bot
   - 时间戳与消息 ID 完整可追踪

验收通过标准:
- 连续 1 次全员回复成功 + 记录消息 ID

---

## 五、团队协作规范(强烈建议)

- 沟通指令必须真实 mention 责任人。
- 需要多人动作时,明确列出人名 + 回执格式。
- 不被 mention 不默认需要回复。
- 验收消息保留 message ID,便于追溯。

---

## 六、一句话复盘

> Discord 多 Agent 协作里，最常见问题不是"模型不聪明"，而是"消息触发条件不成立"。
> 把 **真实 mention + allowBots + requireMention + intent** 四件事配正确，链路就稳。

---

## 七、`ignoreOtherMentions` 陷阱（2026-04-01 新增）

### 现象

- `pegasus`（main bot）在 `#project-pegasus` 频道发了含 `<@Dev_ID>` 的消息。
- Dev 没有任何响应。
- 日志显示：`"reason":"no-mention" discord: skipping guild message`。
- 但消息里确实有真实 mention，`mentions[]` 非空。

### 根因

Dev / QA / Doctor 三个账号均配置了：

```json
"guilds": {
  "*": {
    "requireMention": true,
    "ignoreOtherMentions": true
  }
}
```

`ignoreOtherMentions: true` 的含义是：**只处理"消息内容是直接指令给我本 bot"的 mention，忽略消息里 mention 到我的其他情况**。

实际行为：bot 收到消息后，检查「这条消息是不是主要在叫我处理」，判断为否 → skip。
结果：即使消息里有 `<@DevID>`，只要发送方是另一个 bot 且 `ignoreOtherMentions: true`，Dev 就不会响应。

### 修复

删除所有 agent 账号的 `ignoreOtherMentions` 字段，保留 `requireMention: true` 即可：

```json
// 修复前
"guilds": {
  "*": {
    "requireMention": true,
    "ignoreOtherMentions": true   // ← 删掉这行
  }
}

// 修复后
"guilds": {
  "*": {
    "requireMention": true
  }
}
```

### 影响范围

本次受影响账号：`dev`、`qa`、`doctor`（三个均有此配置）。

### 验证

修复后在 `#project-pegasus` 发 `<@Dev_ID>` 测试消息，Dev 正常回复，日志不再出现 `reason: no-mention`。配置支持热重载，无需重启 gateway。

### 一句话

> `ignoreOtherMentions: true` 会让 bot 在 **被 mention 时仍然不响应**，是 bot-to-bot 协作中最隐蔽的坑之一。除非明确需要只响应人类指令，否则不要设置。
