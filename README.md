# OpenClaw 多 Agent 系统加固实战指南

## 序言

本文基于生产环境实战经验而作——一个运行 6 个 Agent、通过 Telegram Bot 入口、部署在 macOS LaunchAgent 上的 OpenClaw 多 Agent 系统。在这个系统中，我们遭遇了一系列问题：Agent 越权修改核心配置、安全约束在长对话中逐渐失效、关键文件在不知不觉中被篡改、技能目录无限膨胀、视频文件静默丢失……

这些问题看似独立，但共同反映了多 Agent 系统设计中的一个普遍困境：**如何在开放的对话式界面上维持硬性约束**。

本文记录了这些真实问题的完整解决方案。这些方案不是理论抽象，而是经过生产环境验证的实战工程。某些问题（如 Hook 中调用 CLI 的 7 个深坑）在开源社区中鲜有人全面总结；某些解决方案（如 Soul Anchor 的插件化约束注入）需要深入理解 OpenClaw 架构才能实现。

希望这份指南能帮助 OpenClaw 用户构建更加可靠、更加可控的多 Agent 系统。

---

## 一、上下文稀释——最隐蔽的失控

### 问题现象

开发者通常会在 SOUL.md 中写下 Agent 的身份定位、安全约束和工作协议。一切看起来完美。但在实际使用中会发现：**同一个 Agent，在第一条消息中会严格遵守约束，到了第 50 条消息时，约束就逐渐失效了。**

典型表现：
- Agent 被禁止修改 `openclaw.json`，但在长对话后开始尝试修改
- 群聊模式下，Agent 应该静默不发言，但后期会抢话
- 代码修改应该通过 `claude_code` 工具，但后期自行创建新文件

### 根因：Attention U型分布与约束衰减

这不是 OpenClaw 的 bug，而是 LLM 的内在特性。

在 Transformer 模型中，注意力分布遵循**U 型曲线**：
- 对话开头（系统约束所在处）：高注意力
- 对话中间：注意力逐渐衰减
- 对话最后（最新消息）：注意力重新升高

系统约束（SOUL.md）通常放在对话最开头，随着对话深化，模型对这部分内容的注意力逐渐衰减。当对话长度超过某个阈值（通常 20-40 轮），约束的有效性会明显下降。

在多 Agent 系统中，问题更加严重。多个 Agent 的系统约束叠加在 system prompt 中，而 system prompt 本身就是上下文的最前端，形成了"约束的约束"。当总上下文长度足够大时，这层约束极易失效。

### 解决方案：Soul Anchor Plugin

Soul Anchor 的核心思想是**将约束从"一次性注入"改为"持续轮循注入"**。

**关键架构洞察：Hook vs Plugin 的代码路径差异**

OpenClaw 中有两种扩展机制：
- **Managed Hooks**（`~/.openclaw/hooks/`）：简单的事件处理
- **Plugins**（`~/.openclaw/extensions/`）：完整的扩展系统

两者在 `before_prompt_build` 阶段的行为完全不同：

```
Managed Hooks 路径:
  triggerInternalHook()
    → registerInternalHook()
    → fire-and-forget (返回值被忽略)

Plugins 路径:
  hookRunner.runBeforePromptBuild()
    → registry.typedHooks['before_prompt_build']
    → 收集所有 appendSystemContext 返回值
    → 将它们追加到最终的 system prompt
```

**这意味着：只有 Plugin 的 `before_prompt_build` 返回值才会被真正使用。**

Soul Anchor 利用这一点，通过 Plugin 机制在**每一轮对话建立之前**，将硬性约束追加到 system prompt 的末尾（而非前端）。由于是最后追加，这些约束在对话构建时离 LLM input 最近，注意力衰减最小。

**实现细节**

```javascript
// ~/.openclaw/extensions/soul-anchor/index.js
api.on('before_prompt_build', async (context) => {
  const soulAnchorPath = path.join(workspacePath, 'SOUL-ANCHOR.md');
  const soulContent = await fs.promises.readFile(soulAnchorPath, 'utf8');

  return {
    appendSystemContext: soulContent
  };
});
```

这样，每次 LLM 接收新消息时，SOUL-ANCHOR.md 的内容都会被追加到 system prompt 末尾，成为模型最后看到的指令。

**约束文件位置**

```
~/.openclaw/workspaces/
├── main/SOUL-ANCHOR.md              # StudioBot
├── mctech_hr/SOUL-ANCHOR.md         # 马伯乐（HR Agent）
├── mctech_pm_ops/SOUL-ANCHOR.md     # 马全能（PM Agent）
├── mctech_dev_qian/SOUL-ANCHOR.md   # 马前蹄（开发 Agent 1）
└── mctech_dev_hou/SOUL-ANCHOR.md    # 马后炮（开发 Agent 2）
```

**约束内容示例**

SOUL-ANCHOR.md 的典型结构：

```markdown
# 身份边界与核心约束

## 禁止事项（显式硬约束）

**禁止 1：不能修改 openclaw.json**
- 如果需要修改配置，必须通知用户手动修改
- 禁止任何形式的 JSON 直接编辑、替换或写入操作
- 这包括通过 file_write、编程语言脚本等任何方式

**禁止 2：不能直接写入 ~/.openclaw/skills/** 目录
- 所有新 skill 申请必须提交到 skill-requests.md
- 等待 mctech_dev_hou 的审核和开发

## 功能约束

### Workspace 文件规范
- 只能在本 workspace 目录内创建文件
- 禁止跨 workspace 访问其他 Agent 的私有目录

### 群聊静默协议
- 在 Telegram 群聊中收到消息时，仅分析、不主动回复
- 除非被 @ 或明确提问，否则保持静默

## 技能提醒

**如果你是开发 Agent（马前蹄或马后炮）：**
代码修改必须使用 `claude_code` 工具，不能直接 file_write。
允许使用 `claude --dangerously-skip-permissions` 进行代码格式化和自动化任务。

**其他 Agent：**
禁止使用 `--dangerously-skip-permissions` 标志。

## 长任务协议

所有 Agent（包括开发 Agent）：
- 发现任务需要 30 秒以上时，必须通知用户使用异步脚本
- 不能直接堵塞对话等待长任务完成
- 提供 cron job 或 scheduled task 的方案

## Skill 申请协议

当发现缺少某项能力时：

1. 在本地搜索是否已有类似 skill
2. 检查 skill-requests.md 是否已有同类申请（避免重复）
3. 如果没有，追加新申请：
   ```
   ### [待审核] skill-name
   - 申请人：你的 Agent 名称
   - 用途：简明描述
   - 示例代码：粘贴一个使用示例
   - 申请时间：YYYY-MM-DD HH:MM
   ```
4. 等待 mctech_dev_hou 的处理（通常 24-48 小时）

---

## 身份边界（对每个 Agent 需要定制）

[这部分在每个 workspace 的 SOUL-ANCHOR.md 中单独定制]

### 对 mctech_dev_qian（马前蹄）的约束

**身份：**
代码开发和测试 Agent。只负责编写代码和本地测试。

**越界案例：**
- 写 SOUL-ANCHOR.md（这是系统架构，不是代码）
- 修改其他 workspace 的配置
- 直接写入 skills/ 目录
- 修改 openclaw.json

**后发现：**
我们在 2026-03-12 发现，即使加了"禁止"约束，马前蹄仍然在第 45 轮对话中开始尝试修改 SOUL-ANCHOR.md。
**解决：** 在"身份边界"一级加上显式禁令：
```
# 马前蹄（mctech_dev_qian）是代码开发 Agent
# 不能编辑任何架构配置文件（SOUL-ANCHOR.md, openclaw.json 等）
# 不能修改其他 Agent 的 workspace
```

这样在对话开头就明确拒绝，结合末尾的约束注入，形成"首尾呼应"的防护。
```

### 约束生效时间

修改 SOUL-ANCHOR.md 后，约 60 秒内自动生效（Plugin 有内部缓存 TTL）。不需要重启 gateway。

---

## 二、配置文件保护——三重防线

### 问题背景

OpenClaw 的 `openclaw.json` 是整个系统的核心配置：workspace 列表、hook 定义、扩展清单、Telegram 连接参数等。如果被意外修改或恶意篡改，整个系统会崩溃。

但在多 Agent 系统中，任何 Agent 都有文件访问能力，都可能通过 LLM 推理修改这个文件。尤其是在长对话、约束衰减的情况下，风险极高。

### 解决方案：分层防线

#### 第一层：文件系统级 Immutable

**macOS:**
```bash
chflags uchg ~/.openclaw/openclaw.json
```

**Linux:**
```bash
chattr +i ~/.openclaw/openclaw.json
```

**效果：** 即使有 root 权限，也无法删除或修改这个文件（只有 `chflags nouchg` 或 `chattr -i` 才能解除）。

**验证：**
```bash
# macOS
ls -lO ~/.openclaw/openclaw.json
# 应该看到 uchg 标志

# Linux
lsattr ~/.openclaw/openclaw.json
# 应该看到 i------- 标志
```

#### 第二层：SOUL-ANCHOR 软约束

在每个 Agent 的 SOUL-ANCHOR.md 中明确写入：

```markdown
## 禁止修改 openclaw.json

openclaw.json 是系统核心配置文件，你不能修改它。

如果系统需要配置变更：
1. 通知用户
2. 提示用户手动修改
3. 用户修改后重启 gateway

禁止的操作：
- fs.writeFileSync / fs.writeFile 直接写入
- JSON.stringify 替换整个文件
- sed / awk / jq 等命令行工具修改
- 任何形式的文件编辑
```

这一层面向 LLM 的语义理解。在约束充分的情况下，LLM 会避免这种操作。

#### 第三层：Soul Anchor 持续注入

通过 Soul Anchor Plugin，这条约束在每轮对话前都被重新注入到 system prompt 末尾，确保即使约束衰减，也在最后时刻被再次提醒。

### 修改流程

当真的需要修改 openclaw.json 时：

```bash
# 1. 解锁
chflags nouchg ~/.openclaw/openclaw.json  # macOS
chattr -i ~/.openclaw/openclaw.json       # Linux

# 2. 修改（人工操作）
vim ~/.openclaw/openclaw.json
# 或通过其他编辑器修改

# 3. 重新锁定
chflags uchg ~/.openclaw/openclaw.json    # macOS
chattr +i ~/.openclaw/openclaw.json       # Linux

# 4. 重启 gateway
openclaw gateway restart
# 或根据实际启动方式重启
```

### 监控

建议定期检查文件权限状态：

```bash
# 创建一个 cron job
0 9 * * * launchctl list | grep openclaw && \
  ( [ $(chflags -L ~/.openclaw/openclaw.json | grep -c uchg) -gt 0 ] && \
    echo "✓ openclaw.json immutable" || \
    echo "⚠️ openclaw.json mutable!" )
```

---

## 三、Skill 管控——申请制防止能力膨胀

### 问题现象

在多 Agent 系统中，每个 Agent 都可能发现自己"缺少某项能力"。一个常见的做法是 Agent 自行创建 skill——写 SKILL.md、定义输入输出、实现逻辑。

短期看，这提高了灵活性。但长期会导致：
- **质量不可控**：Agent 创建的 skill 可能有 bug、缺少错误处理、性能差
- **目录膨胀**：6 个 Agent 各自创建 5-10 个 skill，很快就有数十个；其中一些只用过一两次
- **重复开发**：多个 Agent 可能各自创建相似的 skill，浪费资源
- **维护困难**：没人清理僵尸 skill，版本不统一

### 方案架构

实现一个 **申请制 + 工作流** 系统，由专职开发 Agent（马后炮）负责开发，其他 Agent 通过申请触发。

```
┌─────────────────┐
│  其他 Agent     │
│  发现缺技能     │
└────────┬────────┘
         │ 提交申请 (skill-requests.md)
         ↓
┌─────────────────────────────────┐
│   skill-requests.md 队列        │
│  [待审核] skill-xxx             │
│  [开发中] skill-yyy             │
│  [测试中] skill-zzz             │
│  [已完成] skill-aaa             │
└────────┬────────────────────────┘
         │ 马后炮 Cron 检查
         ↓
┌─────────────────────────────────┐
│   mctech_dev_hou (马后炮)       │
│   - 查重（确认无重复）           │
│   - 评估（值不值得做）           │
│   - 开发（写 SKILL.md）         │
│   - 测试（在本地运行）           │
│   - 部署（cp 到 ~/.openclaw/skills）
└─────────────────────────────────┘
```

### 实现细节

#### 约束部分

在每个 Agent 的 SOUL-ANCHOR.md 中加入：

```markdown
## 技能目录保护 & 申请制流程

### 禁止直接写入 skills/ 目录

你不能创建或修改以下目录中的任何内容：
- `~/.openclaw/skills/` —— 系统 skill 库
- 当前 workspace 下的 `skills/` 目录

这包括：
- 创建新的 SKILL.md 文件
- 修改现有 skill 的实现
- 删除 skill（即使是你之前创建的）

### 为什么这样设计？

多 Agent 自由创建 skill 会导致：
1. **质量不可控** —— 没有 code review
2. **能力膨胀** —— 关键功能和一次性工具混在一起
3. **维护成本高** —— 没人清理僵尸 skill

### Skill 申请流程

**当你发现需要某个能力时：**

1. 检查现有 skill（不要重复申请）
   ```bash
   ls -la ~/.openclaw/skills/ | grep -i keyword
   ```

2. 搜索 skill-requests.md，看是否已有同类申请
   ```bash
   grep -i keyword ~/.openclaw/shared/skill-requests.md
   ```

3. 如果都没有，在 skill-requests.md 末尾追加：
   ```markdown
   ### [待审核] skill-csv-parser
   - 申请人：mctech_pm_ops（马全能）
   - 用途：解析 Telegram 上传的 CSV，转换为 JSON 表格
   - 示例调用：
     ```
     csv-parser --input data.csv --output json
     ```
   - 申请时间：2026-03-15 14:30
   ```

4. **然后等待**。马后炮会定期检查 skill-requests.md，按优先级开发。通常 24-48 小时内会有反馈。

### Skill 申请状态流转

```
[待审核] → 查重、评估 → [开发中] → 代码实现 → [测试中] → 用户反馈 → [已完成]
                                                              ↓
                                                       [已拒绝] / [已阻塞]
```

状态说明：
- **待审核**：刚提交，等待马后炮审看
- **开发中**：马后炮已认可，正在开发
- **测试中**：代码完成，邀请申请者测试反馈
- **已完成**：测试通过，skill 已部署到 `~/.openclaw/skills/`
- **已拒绝**：评估后认为不值得做，或功能重叠
- **已阻塞**：因为某个上游依赖或设计问题暂停

#### Cron 驱动

马后炮配置定期任务：

```bash
# ~/.openclaw/workspaces/mctech_dev_hou/crontab
# 白天每 30 分钟检查一次 skill-requests.md
07,37 9-20 * * * /usr/local/bin/skill-request-cron.sh

# 晚上每 3 小时检查一次
13 21,0,3,6 * * * /usr/local/bin/skill-request-cron.sh
```

cron 脚本的伪代码：

```bash
#!/bin/bash
# skill-request-cron.sh

SKILL_REQUESTS="$HOME/.openclaw/shared/skill-requests.md"

# 1. 检查有没有 [待审核] 项
if grep -q "^\### \[待审核\]" "$SKILL_REQUESTS"; then
  # 2. 抽取第一个待审核项
  SKILL_NAME=$(grep "^\### \[待审核\]" "$SKILL_REQUESTS" | head -1 | sed 's/.*\] //g')

  # 3. 启动一个 Claude Code 会话
  # 在 SOUL-ANCHOR.md 中提示"有新 skill 申请待处理"

  # 4. 更新状态为 [开发中]
  sed -i.bak "s/\[待审核\] $SKILL_NAME/[开发中] $SKILL_NAME/" "$SKILL_REQUESTS"
fi
```

### 为什么选择软约束而非硬拦截？

这是一个架构选择，需要解释：

**硬拦截（路径级权限控制）的问题：**
- OpenClaw 没有原生的 RBAC 系统
- 即使限制目录访问，LLM 可能通过其他方式绕过（如 symlink、环境变量注入）
- 难以处理"同一个 Agent 在某些情况下允许写入"的场景（如开发 Agent）

**软约束（SOUL-ANCHOR 指导 + 人工工作流）的优势：**
- 与 LLM 的语义理解对齐，不需要复杂的系统权限
- 灵活性高，可以根据具体情况调整策略
- 搭配持续的约束注入（Soul Anchor），可以有效降低越权风险
- 即使 Agent 试图越权，也会留下清晰的操作日志（attempt to modify files）

**成本：**
- 需要额外的人工审核（马后炮的定期检查）
- 需要 Cron 驱动来保证及时性

这个权衡对大多数团队来说是合理的。

### 关键教训

**教训 1：约束首尾呼应**

一开始我们只在 SOUL-ANCHOR.md 中加"禁止写入 skills/ 目录"，但在第 45 轮对话中，马前蹄仍然尝试创建 skill。

**原因：** 约束在对话中间衰减了。

**解决：** 在 SOUL-ANCHOR.md 的最前面（身份边界一级）也加上显式禁令：

```markdown
# 身份与职责

## mctech_dev_qian（马前蹄）

**职责：** 后端代码开发和测试

**禁止事项：**
- 不能修改架构配置文件（openclaw.json, SOUL-ANCHOR.md）
- 不能写入 ~/.openclaw/skills/ 目录
- 不能修改其他 Agent 的 workspace

[其他约束...]
```

这样在对话最开头就清晰地标记，结合末尾的约束注入，形成双层防护。

**教训 2：申请文件的单一来源**

skill-requests.md 是所有 Agent 的共享队列，这带来并发问题：如果两个 Agent 同时修改，可能产生冲突。

**解决：** 使用 Git 的 auto-merge 特性 + 定期手工整理。或者迁移到数据库（但这对小团队过度工程化）。

---

## 四、会话存储膨胀——看不见的性能杀手

### 问题发现

在运行 6 个 Agent 的系统中，OpenClaw 的会话存储（SQLite 或类似）逐渐膨胀。每个对话会记录完整的消息历史、token 使用、模型回复等。

一个表面看似"10 轮对话"的会话，实际占用的存储空间可能是预期的 2-3 倍。原因有两个：

**问题 1：Thinking Blocks**

在使用 Claude 3.7+ 模型时，如果启用了 extended thinking（长思考模式），模型会在内部生成大量的思考过程。这些 thinking blocks 在序列化时占用约 50% 的 token 空间，但对最终回复没有帮助——LLM 不会在后续对话中回看这些 thinking blocks。

**问题 2：软删除未清理**

当用户删除消息时，OpenClaw 通常执行"软删除"（标记为 deleted，但不真正删除数据库记录）。长期积累下来，软删除的记录占用大量空间。

### 具体数据

在一个有 6 个 Agent、运行 3 个月的系统中：

```
清理前：
- 总会话存储：156.3 MB
- 活跃消息：~8,500 条
- Thinking blocks：78.2 MB（50.0%）
- 软删除记录：34.1 MB（21.8%）
- 其他（metadata 等）：43.9 MB（28.1%）

执行清理后：
- 总会话存储：131.7 MB
- 释放空间：24.6 MB（15.8%）
```

虽然相对比例不算极端，但在高频对话的系统中，这个数字会迅速增长。

### 解决方案：Context Cleaner

Context Cleaner 是一个 OpenClaw Hook + Cron 脚本的组合方案。

**工作原理：**

1. **Hook 层**（`~/.openclaw/hooks/context-cleaner/handler.js`）：
   - 监听 `after_message_processed` 事件
   - 检查当前会话的 token 使用量
   - 如果超过阈值（如 80,000 tokens），标记会话为"需要清理"

2. **Cron 层**（定期任务）：
   - 每天凌晨 2 点执行
   - 扫描所有标记为"需要清理"的会话
   - 删除 thinking blocks 和软删除记录
   - 重新 pack 数据库（如果使用 SQLite）

**实现伪代码：**

```javascript
// hook handler.js
api.on('after_message_processed', async (context) => {
  const { sessionId, tokenUsage } = context;

  // 如果这轮对话的累计 token > 80,000
  if (tokenUsage.total > 80000) {
    // 标记会话
    await markSessionForCleaning(sessionId);

    // 记录日志（便于后续分析）
    logger.info(`Session ${sessionId} marked for cleaning: ${tokenUsage.total} tokens`);
  }
});
```

```bash
#!/bin/bash
# cleanup-cron.sh 每天 02:00 执行

DB_PATH="$HOME/.openclaw/sessions.db"

# 1. 删除所有 thinking_blocks 记录
sqlite3 "$DB_PATH" "DELETE FROM message_content WHERE type = 'thinking_block';"

# 2. 物理删除软删除的消息
sqlite3 "$DB_PATH" "DELETE FROM messages WHERE deleted_at IS NOT NULL;"

# 3. 重建索引和清理碎片
sqlite3 "$DB_PATH" "VACUUM; REINDEX;"

echo "Context cleanup completed at $(date)" >> ~/.openclaw/cron-logs/context-cleaner.log
```

**配置示例：**

```bash
# 在 LaunchAgent plist 中添加定期执行
# ~/.openclaw/LaunchAgents/ai.openclaw.context-cleaner.plist

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>ai.openclaw.context-cleaner</string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>/Users/eric/.openclaw/crons/context-cleaner.sh</string>
    </array>
    <key>StartCalendarInterval</key>
    <dict>
        <key>Hour</key>
        <integer>2</integer>
        <key>Minute</key>
        <integer>0</integer>
    </dict>
    <key>StandardErrorPath</key>
    <string>/Users/eric/.openclaw/cron-logs/context-cleaner.err</string>
    <key>StandardOutPath</key>
    <string>/Users/eric/.openclaw/cron-logs/context-cleaner.out</string>
</dict>
</plist>
```

**详细文档参见：** `openclaw-context-cleaner` 项目

---

## 五、在 Hook 中调用外部 CLI——7 个深坑

### 背景

OpenClaw Hook 本质上是在 Node.js 事件循环中执行的 JavaScript。当需要调用外部工具（如 Claude Code CLI `claude`）时，需要使用 `child_process` 模块。这听起来简单，但实际上有 7 个深层坑，每一个都可能导致生产事故。

### 坑 1：Hook 放置位置（Workspace vs Global）

**现象：**

一开始我们把 `tg-proxy` hook 放在 `~/.openclaw/hooks/tg-proxy/`（全局 hooks）。但在多 workspace 环境下，这个 hook 会被所有 workspace 共享调用，导致消息路由混乱。

**根因：**

OpenClaw 的 Hook 系统有两个层级：
- **Global hooks**（`~/.openclaw/hooks/`）：所有 workspace 都会触发
- **Workspace hooks**（`~/.openclaw/workspaces/{workspace}/hooks/`）：仅该 workspace 触发

如果 tg-proxy 在全局放置，Telegram 消息会被所有 workspace 的 agent 同时处理，产生竞态条件。

**解决：**

根据消息来源的 workspace 信息，在 hook 中显式路由。或者，对于专属于某个 Agent 的 hook，放在该 workspace 下：

```
~/.openclaw/workspaces/main/hooks/tg-proxy/handler.js
```

**最佳实践：** 优先使用 workspace-level hooks，除非确实需要全局行为。

### 坑 2：ACPX 协议崩溃，改用 `claude -p`

**现象：**

早期，我们通过 `claude --api-key xxx --project yyy --cwd zzz` 直接调用 Claude Code 的 ACPX 协议。但发现在 Hook 中调用时，ACPX 协议会随机崩溃，错误信息含糊不清。

**根因：**

ACPX 是 Claude Code 的内部协议，设计用于 IDE 集成，不是为了被外部进程频繁调用。在高并发场景下（多个 hook 同时触发），协议容易出现竞态条件。

**解决：**

改用 `claude -p`（Project Mode）。这是 Claude Code 的标准 CLI 模式，更加健壮：

```bash
# 坏例子：
claude --api-key sk-xxx --project yyy --cwd zzz "analyze this code"

# 好例子：
echo "analyze this code" | claude -p
```

**为什么 `-p` 更稳定？**

`-p` 模式会自动从 `~/.openclaw/claude.config` 读取项目信息，而不依赖命令行参数。它也是 Claude Code 团队重点测试和维护的模式。

### 坑 3：macOS Keychain 需要 USER 环境变量

**现象：**

在 macOS LaunchAgent 中启动 OpenClaw，Hook 调用 `claude -p` 时，会随机遇到"Keychain 访问失败"错误。而同一个命令在终端手工运行时完全正常。

**根因分析：**

Claude Code 在启动时需要从 macOS Keychain 中读取 OAuth token（用于 Anthropic API）。Keychain 的权限验证依赖于 `USER` 和 `LOGNAME` 环境变量，用来确认"是哪个用户在请求访问"。

LaunchAgent 的问题：
```xml
<!-- LaunchAgent plist 中，环境变量传递有限 -->
<key>EnvironmentVariables</key>
<dict>
    <!-- 默认只有 PATH, HOME 等少数变量 -->
</dict>
```

当 Hook 中的 Node.js 进程 spawn 一个子进程执行 `claude`，如果没有显式传递 `USER` 和 `LOGNAME`，Keychain 会拒绝访问。

**解决：**

在 hook handler 中，显式传递必要的环境变量：

```javascript
const { spawn } = require('child_process');

const env = {
  ...process.env,
  USER: process.env.USER || 'eric',      // ← 关键
  LOGNAME: process.env.LOGNAME || 'eric', // ← 关键
};

const proc = spawn('claude', ['-p'], {
  env: env,
  cwd: workspacePath,
});
```

**验证方法：**

```bash
# 查看当前环境变量
env | grep -E 'USER|LOGNAME'

# 在 LaunchAgent 中补充声明
defaults write ~/Library/LaunchAgents/ai.openclaw.gateway \
  EnvironmentVariables -dict USER eric LOGNAME eric
```

### 坑 4：process.env 污染子进程

**现象：**

Hook 调用 `claude -p`，会继承 OpenClaw gateway 进程的所有环境变量。这包括：
- OpenClaw 内部的 `OPENCLAW_*` 变量
- Gateway 的 `LOG_LEVEL`, `DEBUG_MODE` 等
- 某些变量（如 `HTTP_PROXY`, `HTTPS_PROXY`）可能导致 Claude Code 使用错误的代理

结果：Hook 调用的 Claude 表现异常。

**根因：**

Node.js 的 `spawn()` 默认继承当前进程的所有 `process.env`。如果 parent 进程的环境被"污染"，child 进程会直接继承。

**解决：**

使用"清洁环境"原则，只传递必需的变量：

```javascript
const cleanEnv = {
  // 必需的系统变量
  PATH: process.env.PATH,
  HOME: process.env.HOME,
  USER: process.env.USER,
  LOGNAME: process.env.LOGNAME,

  // 必需的代码执行环境
  NODE_ENV: 'production',

  // 不要包含：OPENCLAW_*, LOG_LEVEL, DEBUG_MODE 等
};

const proc = spawn('claude', ['-p'], {
  env: cleanEnv,
  cwd: workspacePath,
});
```

**最小环境原则：** 只在 spawn 时显式指定必需的环境变量，而非继承整个 process.env。

### 坑 5：zsh -l 在非 TTY 加载 Oh My Zsh 阻塞

**现象：**

我们的 Hook 为了确保 shell 正确加载用户配置（`.zshrc`, `.bashrc`），使用了：

```javascript
const proc = spawn('zsh', ['-l', '-c', 'claude -p'], ...);
```

参数 `-l`（login shell）会使 zsh 加载完整的初始化脚本。这在 LaunchAgent（非 TTY）中会导致长达 10-15 秒的阻塞，因为 Oh My Zsh 会尝试加载 theme、plugin 等。

**根因：**

zsh 在 non-TTY 环境下的行为不同：
- TTY 环境：可以交互，加载快速
- non-TTY 环境（如 LaunchAgent、cron、Hook）：会阻塞等待初始化完成

**解决：**

不要通过 shell 包装。直接 spawn Claude Code：

```javascript
// 坏例子：
spawn('zsh', ['-l', '-c', 'claude -p'], ...)

// 好例子：
spawn('claude', ['-p'], ...)
```

Claude Code 自身会读取必要的环境变量，无需 shell 包装。

**如果一定要 shell：**

使用 `--noprofile --nointrc` 禁用初始化脚本加载：

```javascript
spawn('zsh', ['--noprofile', '--nointrc', '-c', 'claude -p'], ...)
```

但最佳实践仍然是不要用 shell 中间件。

### 坑 6：execFile 不关闭 stdin 导致 claude -p 永久挂起（终极坑）

**现象：**

一开始我们使用 Node.js 的 `execFile()` 来调用 Claude Code：

```javascript
const { execFile } = require('child_process');

execFile('claude', ['-p'], {
  timeout: 30000,
  cwd: workspacePath,
  input: userMessage,  // 传递输入
}, (error, stdout, stderr) => {
  // 处理结果
});
```

问题：约 20% 的情况下，`claude -p` 会永久挂起，Hook 最终 timeout 失败。

**根因分析（深度）：**

这涉及 Claude Code 的 stdin 处理逻辑。`claude -p` 期望从 stdin 读取用户输入，直到 **EOF（End of File）** 信号。流程如下：

```
1. claude -p 启动，打开 stdin 监听
2. 读取用户消息：while (stdin.readable) { chunk = read(stdin) }
3. 等待 stdin EOF
4. 收到 EOF → 处理完整消息 → 生成回复
```

`execFile()` 的问题：
```javascript
execFile(cmd, args, {
  input: data,  // ← 注意：这个 input 参数的处理
  ...
})
```

当你使用 `input` 参数时，Node.js 会：
1. 将数据写入 stdin
2. **但不一定立即关闭 stdin**（这取决于 Node.js 版本和系统状态）

如果 stdin 没有被立即关闭，Claude Code 仍然在等待更多输入。由于对面是 Hook 进程（not TTY），Claude Code 不会超时（等待 TTY 用户输入），最终 Hook 的 timeout 被触发，杀死子进程。

**解决方案：使用 spawn + 显式关闭 stdin**

```javascript
const { spawn } = require('child_process');

const proc = spawn('claude', ['-p'], {
  stdio: ['pipe', 'pipe', 'pipe'],
  timeout: 30000,
  cwd: workspacePath,
});

// 写入用户消息
proc.stdin.write(userMessage);

// 关键：立即关闭 stdin，发送 EOF 信号
proc.stdin.end();

// 监听输出
let output = '';
proc.stdout.on('data', (chunk) => {
  output += chunk.toString();
});

proc.on('close', (code) => {
  if (code === 0) {
    // 成功
    callback(null, output);
  } else {
    callback(new Error(`claude exited with code ${code}`));
  }
});
```

**关键差异：**

```javascript
// 坏：execFile
execFile('claude', ['-p'], { input: msg }, callback);
// Node.js 负责写入和关闭，时序不确定

// 好：spawn + stdin.end()
const proc = spawn('claude', ['-p']);
proc.stdin.write(msg);
proc.stdin.end();  // ← 显式发送 EOF
proc.on('close', callback);
```

通过显式调用 `proc.stdin.end()`，我们保证 stdin 会立即关闭，Claude Code 能正确识别输入完毕。

**验证：**

```bash
# 测试 CLI 直接调用
echo "hello world" | claude -p
# ✓ 应该立即完成

# 如果卡住，说明 stdin 没有被关闭
echo "hello world" | timeout 5 claude -p
# 如果 timeout 触发，说明有问题
```

### 坑 7：execFile timeout 杀父不杀子

**现象：**

Hook 设置了 `timeout: 30000`，期望 30 秒后若 Claude Code 未返回就强制杀死。但实际操作中，Hook 进程被杀死了，但子进程（claude 进程）仍然继续运行，变成孤儿进程。

**根因：**

Node.js 的 `timeout` 只会杀死直接的 child 进程，不会递归杀死其 descendants。如果 Claude Code 内部创建了更多子进程（如文件编辑、代码执行等），这些 descendants 不会被清理。

```
Hook Process
  ├─ claude 进程 ← timeout 触发，杀死
  │  ├─ node worker ← 孤儿进程，继续运行
  │  ├─ bash subprocess ← 孤儿进程，继续运行
  │  └─ ...
```

**解决：**

使用进程组（Process Group）来确保 timeout 时杀死整个进程树：

```javascript
const { spawn } = require('child_process');

const proc = spawn('claude', ['-p'], {
  stdio: ['pipe', 'pipe', 'pipe'],
  timeout: 30000,
  detached: true,  // ← 创建独立的进程组
});

proc.stdin.write(userMessage);
proc.stdin.end();

let output = '';
proc.stdout.on('data', (chunk) => {
  output += chunk.toString();
});

proc.on('close', (code) => {
  callback(null, output);
});

// timeout 处理：杀死整个进程组
proc.on('error', (err) => {
  if (err.code === 'ERR_CHILD_PROCESS_TIMEOUT') {
    // 杀死进程组（包括所有子进程）
    process.kill(-proc.pid, 'SIGTERM');  // ← 负号表示进程组

    // 给点时间让进程优雅退出
    setTimeout(() => {
      process.kill(-proc.pid, 'SIGKILL');
    }, 5000);
  }
  callback(err);
});
```

**关键参数：**

```javascript
spawn(cmd, args, {
  detached: true,  // 创建新的进程组
  // 或使用 setsid()(Unix 特定）
})

// 杀死进程组：
process.kill(-processId, signal);  // ← 负号 = 进程组
```

### 总结：完整的生产级实现

```javascript
// ~/.openclaw/hooks/tg-proxy/handler.js - 完整版本
const { spawn } = require('child_process');
const path = require('path');
const logger = require('@openclaw/logger');

async function invokeClaudeCode(userMessage, workspace) {
  return new Promise((resolve, reject) => {
    const workspacePath = path.join(
      process.env.HOME,
      '.openclaw',
      'workspaces',
      workspace
    );

    // 清洁环境
    const cleanEnv = {
      PATH: process.env.PATH,
      HOME: process.env.HOME,
      USER: process.env.USER || 'eric',
      LOGNAME: process.env.LOGNAME || 'eric',
      NODE_ENV: 'production',
    };

    // spawn，不用 shell，创建进程组
    const proc = spawn('claude', ['-p'], {
      stdio: ['pipe', 'pipe', 'pipe'],
      cwd: workspacePath,
      env: cleanEnv,
      detached: true,  // 进程组
      timeout: 30000,
    });

    let output = '';
    let errorOutput = '';

    // 写入消息，立即关闭 stdin
    proc.stdin.write(userMessage);
    proc.stdin.end();

    // 读取输出
    proc.stdout.on('data', (chunk) => {
      output += chunk.toString();
    });

    proc.stderr.on('data', (chunk) => {
      errorOutput += chunk.toString();
      logger.warn(`claude stderr: ${chunk}`);
    });

    // 成功关闭
    proc.on('close', (code) => {
      if (code === 0) {
        resolve({ output, code });
      } else {
        reject(new Error(`claude exited with code ${code}: ${errorOutput}`));
      }
    });

    // 错误处理
    proc.on('error', (err) => {
      logger.error(`spawn error: ${err.message}`);

      // 尝试杀死进程组
      try {
        process.kill(-proc.pid, 'SIGTERM');
        setTimeout(() => {
          process.kill(-proc.pid, 'SIGKILL');
        }, 5000);
      } catch (e) {
        // 进程可能已经退出
      }

      reject(err);
    });

    // timeout 处理
    const timeoutHandle = setTimeout(() => {
      logger.warn(`claude process timeout, killing process group ${proc.pid}`);
      try {
        process.kill(-proc.pid, 'SIGTERM');
        setTimeout(() => {
          process.kill(-proc.pid, 'SIGKILL');
        }, 5000);
      } catch (e) {
        // 已退出
      }
      reject(new Error('Claude process timeout'));
    }, 30000);

    proc.on('close', () => {
      clearTimeout(timeoutHandle);
    });
  });
}

module.exports = { invokeClaudeCode };
```

---

## 六、Telegram 大文件下载——静默失败的黑洞

### 问题背景

OpenClaw 支持通过 Telegram Bot 接收用户上传的文件。视频日记等功能依赖于接收和处理用户的视频文件。但 Telegram Bot API 有一个硬性限制：

**Bot API 一次获取的文件不超过 20 MB。**

用户上传的视频通常 50-200 MB，远超这个限制。更糟的是，OpenClaw gateway 在下载失败时：
1. 用 `logVerbose` 记录错误（只在 debug 模式可见）
2. 静默跳过文件处理
3. 不通知 Agent，不记录到主日志

结果：**文件丢了都不知道**。

### 三层限制链

```
┌────────────────────────────────────┐
│    Telegram Bot API (官方)         │
│  getFile 返回: file_path, file_id  │
│  限制：文件内容 ≤ 20 MB            │
│  (针对 bot getFile 端点，不是服务器存储)
└───────────────┬────────────────────┘
                │ (无法直接下载超大文件)
                ↓
┌────────────────────────────────────┐
│   OpenClaw Gateway                 │
│   - 调用 API.getFile()             │
│   - 尝试下载文件到内存 (Buffer)    │
│   - 文件 > 20MB → 失败             │
│   - logVerbose 记录 + 静默跳过     │
└───────────────┬────────────────────┘
                │ (文件丢失)
                ↓
┌────────────────────────────────────┐
│   Agent / Skill                    │
│   - 收不到文件通知                 │
│   - 被动地得到"没有文件"的假象    │
└────────────────────────────────────┘
```

**问题的关键：**
- Gateway 用 `logVerbose` 记录（生产环境看不到）
- Gateway 在内存中尝试读整个文件（效率差）
- Gateway 不区分"文件不存在"和"文件太大"

### 完整限制详解

**限制 1：Telegram API 官方**

```
Bot API Method: getFile
Input: file_id
Output:
{
  "ok": true,
  "result": {
    "file_id": "...",
    "file_unique_id": "...",
    "file_size": 156482432,    // ← 文件大小（可以是任意大）
    "file_path": "video/..."   // ← 相对路径
  }
}

但如果直接访问 https://api.telegram.org/file/bot{token}/{file_path}
且文件 > 20 MB → 连接重置
```

**限制 2：OpenClaw Gateway 代码**

```javascript
// (伪代码，基于观察)
async function downloadMediaFile(fileId) {
  try {
    const fileInfo = await api.getFile(fileId);
    const fileUrl = `https://api.telegram.org/file/...`;

    // ← 问题：试图将整个文件读入内存
    const buffer = await fetch(fileUrl).then(r => r.buffer());

    // > 20 MB 会失败
    return buffer;
  } catch (err) {
    logger.logVerbose(`Failed to download file: ${err.message}`);
    // ← 静默失败，不抛异常
    return null;
  }
}
```

**限制 3：Telegram Local Server 的 bypass**

Telegram 官方提供了一个本地服务器：

```
telegram-bot-api v9.5+ 支持 --local 模式
在 local 模式下：
- 绕过 API 调用频率限制
- 绕过 20 MB 文件大小限制
- 可以直接访问本地文件系统
- 支持任意大小的文件
```

这是解决方案的关键。

### 解决方案

#### 第一步：部署 Telegram Bot API Local Server

**安装：**

```bash
# 如果没有安装，从官方 release 下载
wget https://github.com/tdlib/telegram-bot-api/releases/download/v9.5/telegram-bot-api-9.5.0.tar.gz
tar xzf telegram-bot-api-9.5.0.tar.gz
cd telegram-bot-api-9.5.0
mkdir build && cd build
cmake .. && make -j4
cp bin/telegram-bot-api ~/.local/bin/

# 验证
~/.local/bin/telegram-bot-api --version
```

**启动 Local Server：**

```bash
# 创建数据目录
mkdir -p ~/.openclaw/telegram-bot-api-data

# 启动 Local Server（脚本化）
# ~/.openclaw/scripts/start-telegram-local-server.sh
#!/bin/bash
~/.local/bin/telegram-bot-api \
  --local \
  --api-id 123456 \
  --api-hash "abcdef..." \
  --database-dir ~/.openclaw/telegram-bot-api-data \
  --port 8081 \
  --http-stat-port 8082
```

**配置 OpenClaw 使用 Local Server：**

在 `openclaw.json` 中修改 Telegram 配置：

```json
{
  "integrations": {
    "telegram": {
      "botToken": "...",
      "apiRoot": "http://localhost:8081",  // ← 使用 local server
      "apiId": 123456,
      "apiHash": "abcdef..."
    }
  }
}
```

**注意（重要）：OpenClaw 2026.3.12 的限制**

OpenClaw 当前版本在三处硬编码了 `api.telegram.org`：
- gateway 的 API 调用
- media downloader hook
- 某些 webhook 处理

PR #27453 尚未合并，计划在 2026.4 版本支持 `apiRoot` 配置。**暂时的 workaround：使用 /etc/hosts 本地 DNS 重定向**

```bash
# /etc/hosts
127.0.0.1  api.telegram.org
::1        api.telegram.org
```

然后在 local server 启动时监听 443 端口（需要 sudo）。

#### 第二步：Media Downloader Hook

创建 `~/.openclaw/hooks/media-downloader/handler.js` 来检测大文件并通过 Local Server 下载：

```javascript
// handler.js
const fs = require('fs').promises;
const https = require('https');
const path = require('path');

api.on('media:received', async (context) => {
  const { file, mediaPath, fileSize } = context;

  // 如果文件大小超过 15 MB，通过 Local Server 下载
  if (fileSize > 15 * 1024 * 1024) {
    logger.info(`Large file detected: ${fileSize} bytes, using Local Server`);

    const localServerUrl = 'http://localhost:8081';
    const downloadUrl = `${localServerUrl}/file/...`;

    try {
      // 流式下载，不读入内存
      const stream = https.get(downloadUrl);
      const savePath = path.join(
        process.env.HOME,
        '.openclaw',
        'media',
        file.file_id
      );

      stream.pipe(fs.createWriteStream(savePath));

      await new Promise((resolve, reject) => {
        stream.on('end', resolve);
        stream.on('error', reject);
      });

      logger.info(`File downloaded successfully: ${savePath}`);
      context.mediaPath = savePath;  // 更新 mediaPath
    } catch (err) {
      logger.error(`Failed to download from Local Server: ${err.message}`);
      // 记录失败，而不是静默跳过
      context.errors = context.errors || [];
      context.errors.push(`Media download failed: ${err.message}`);
    }
  }
});
```

**关键改进：**
- 使用流式下载（`pipe`），不读入内存
- 明确记录失败原因
- 更新 context，而不是静默跳过
- 流式下载支持任意大小的文件

#### 第三步：Video Diary 修复

OpenClaw 的视频日记 skill 有几个 bug（基于 2026.3 版本）：

**Bug 1：SKILL_DIR 路径错误**

```bash
# 原始代码
SKILL_DIR=$(dirname "$0")
cd "$SKILL_DIR" || exit 1

# 问题：$0 在被 cron 或 hook 调用时，可能是相对路径或符号链接
# 导致 cd 失败

# 修复：使用绝对路径
SKILL_DIR="$HOME/.openclaw/skills/video-diary"
cd "$SKILL_DIR" || { echo "Failed to cd to $SKILL_DIR"; exit 1; }
```

**Bug 2：视频处理超时**

```bash
# 原始代码
timeout 60 ffmpeg -i "$input" -c:v libx264 "$output"

# 问题：视频编码 60 秒通常不够

# 修复：增加 timeout，或使用后台处理
timeout 300 ffmpeg -i "$input" -c:v libx264 "$output" &
FFMPEG_PID=$!
wait $FFMPEG_PID
```

**Bug 3：错误处理缺失**

```bash
# 修复后的 video-diary 脚本
#!/bin/bash
set -euo pipefail

SKILL_DIR="$HOME/.openclaw/skills/video-diary"
LOG_FILE="$HOME/.openclaw/logs/video-diary.log"

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $*" >> "$LOG_FILE"
}

if [ ! -d "$SKILL_DIR" ]; then
  log "ERROR: SKILL_DIR not found: $SKILL_DIR"
  exit 1
fi

cd "$SKILL_DIR" || {
  log "ERROR: Failed to cd to $SKILL_DIR"
  exit 1
}

INPUT="${1:?Missing input file}"
OUTPUT="${2:?Missing output file}"

if [ ! -f "$INPUT" ]; then
  log "ERROR: Input file not found: $INPUT"
  exit 1
fi

log "Processing: $INPUT → $OUTPUT"

if timeout 300 ffmpeg -i "$INPUT" -c:v libx264 -crf 23 "$OUTPUT"; then
  log "SUCCESS: Video processed: $OUTPUT"
else
  log "ERROR: ffmpeg failed with code $?"
  exit 1
fi
```

### 中国特殊情况：api_id 注册需代理

在中国注册 Telegram 的 api_id 时，会遇到网络限制。解决方案：

```bash
# 使用代理注册
https_proxy=https://cheapproxy.net:8080 \
  curl -X POST https://my.telegram.org/auth/send_code \
  -d "phone_number=+86..." \
  -d "...其他参数"
```

或者，从海外 VPS 注册后，直接使用 api_id 和 api_hash 即可（Local Server 不需要再次注册）。

### 监控和告警

添加日志监控，检测媒体下载失败：

```bash
#!/bin/bash
# 每 1 小时检查一次
grep -i "failed to download\|media download failed" \
  ~/.openclaw/logs/* \
  ~/.openclaw/hooks/media-downloader.log \
| tail -20
```

或集成到 ELK / Datadog 等监控系统。

---

## 七、设计哲学总结

### 分层防御优于单点完美

不要追求一个"完美"的技术方案。现实的多 Agent 系统中，多层次的防护往往更有效。

例如，config 文件保护：
- 仅用文件系统权限 → 可被绕过（如 root）
- 仅用 LLM 约束 → 长对话后衰减
- 文件系统 + SOUL-ANCHOR + Soul Anchor Plugin → 三重防线，彼此补充

### 静默失败是最大敌人

Telegram 文件下载的案例说明：**技术上"能处理"失败情况，和"显式记录"失败情况是两个概念**。

OpenClaw gateway 技术上处理了下载失败（捕获异常），但因为只用 `logVerbose` 记录，生产环境看不到，等同于"黑洞"。

**原则：** 所有关键路径的失败都应该有显式的、易于查看的日志和告警。

### 最小环境原则

在 Hook 中调用外部 CLI 时，坑 4 展示了：如果直接继承 `process.env`，parent 的"污染"会传给 child。

**解决：** 显式声明 child 需要的最小环境变量集合，而不是继承整个 process.env。

这个原则也适用于：
- Docker 容器的环境变量声明
- Cron job 的环境设置
- LaunchAgent 的 plist 配置

### 软约束 + 流程管控优于完美硬拦截

技能管控的案例：我们没有实现"绝对禁止 Agent 写入 skills/ 目录"的硬拦截，而是用：
- SOUL-ANCHOR 约束（软，但持续注入）
- skill-requests.md 工作流（流程管控）
- 专职开发 Agent 的 cron 驱动（人工参与）

这样做的好处：
- 灵活性高，遇到例外情况（如开发 Agent 需要临时写入）容易调整
- 成本低，不需要复杂的系统权限架构
- 可审计，每个请求都有记录

### 约束首尾呼应

上下文稀释的防御中，我们在 SOUL-ANCHOR.md 的开头和结尾都写约束。这是因为：
- 开头（身份边界）：在对话最开始就明确立场
- 结尾（Soul Anchor 追加）：在对话每一轮前夕再次提醒

这样即使中间衰减，两端的约束仍然有效。

---

## 八、常见问题 vs 方案对照表

| 问题 | 现象 | 本指南方案 | 关键技术 |
|------|------|----------|---------|
| **上下文稀释** | Agent 在长对话后忽视约束 | Soul Anchor Plugin + 首尾呼应 | Plugin `before_prompt_build` 返回值收集 |
| **配置文件被改** | openclaw.json 被意外修改 | 三重防线：文件系统 immutable + SOUL-ANCHOR + Soul Anchor | `chflags uchg` / `chattr +i` + 持续注入 |
| **Skill 无限膨胀** | 多个 Agent 各自创建 skill，质量不可控 | skill-requests.md 申请制 + 专职开发 + Cron 驱动 | 软约束 + 工作流 + 人工审核 |
| **会话存储膨胀** | 数据库占用逐渐增大，性能下降 | Context Cleaner：删除 thinking blocks + 软删除记录 | Hook 监控 + Cron 清理 + SQLite VACUUM |
| **CLI 调用失败** | Hook 调用 Claude 时随机失败或挂起 | spawn + stdin.end() + 清洁环境 + 进程组 | 7 个坑的完整解决方案 |
| **大文件静默丢失** | 视频上传后文件消失，无人知晓 | Telegram Local Server + media-downloader hook + 显式日志 | Local Server bypass + 流式下载 + 错误追踪 |
| **Agent 越权** | Agent 修改其他 workspace 配置 | SOUL-ANCHOR 身份边界 + 约束首尾呼应 | 显式禁令 + 持续注入 |
| **Keychain 访问失败** | LaunchAgent 中调用 claude 失败 | 显式传递 USER / LOGNAME 环境变量 | `cleanEnv` 最小集合 |

---

## 附录 A：完整的 openclaw.json 配置示例

```json
{
  "version": "2026.3.12",
  "workspaces": [
    {
      "id": "main",
      "name": "StudioBot",
      "description": "Main studio bot",
      "modelConfig": {
        "model": "claude-3-7-sonnet-20250219",
        "maxTokens": 8192
      }
    },
    {
      "id": "mctech_hr",
      "name": "马伯乐 (HR Agent)",
      "description": "Human Resources Agent"
    },
    {
      "id": "mctech_pm_ops",
      "name": "马全能 (PM Agent)",
      "description": "Project Management Agent"
    },
    {
      "id": "mctech_dev_qian",
      "name": "马前蹄 (Dev Agent 1)",
      "description": "Backend Development Agent"
    },
    {
      "id": "mctech_dev_hou",
      "name": "马后炮 (Dev Agent 2)",
      "description": "Infrastructure & DevOps Agent"
    }
  ],
  "integrations": {
    "telegram": {
      "enabled": true,
      "botToken": "YOUR_BOT_TOKEN_HERE",
      "apiRoot": "http://localhost:8081",
      "apiId": 123456,
      "apiHash": "YOUR_API_HASH_HERE",
      "webhookUrl": "https://your-domain.com/telegram-webhook"
    }
  },
  "extensions": [
    {
      "id": "soul-anchor",
      "type": "plugin",
      "path": "~/.openclaw/extensions/soul-anchor",
      "enabled": true
    }
  ],
  "hooks": [
    {
      "id": "tg-proxy",
      "type": "message:preprocessed",
      "path": "~/.openclaw/hooks/tg-proxy/handler.js",
      "enabled": true
    },
    {
      "id": "media-downloader",
      "type": "media:received",
      "path": "~/.openclaw/hooks/media-downloader/handler.js",
      "enabled": true
    },
    {
      "id": "context-cleaner",
      "type": "after_message_processed",
      "path": "~/.openclaw/hooks/context-cleaner/handler.js",
      "enabled": true
    }
  ]
}
```

---

## 附录 B：Soul Anchor 配置检查清单

- [ ] 在 `~/.openclaw/extensions/soul-anchor/` 下创建 Plugin
- [ ] `openclaw.plugin.json` 包含 `configSchema` 字段
- [ ] `index.js` 中使用 `api.on('before_prompt_build', ...)` 注册（不是 hook）
- [ ] 每个 workspace 都有 `SOUL-ANCHOR.md` 文件
- [ ] SOUL-ANCHOR.md 包含：身份边界、禁止事项、功能约束、技能提醒、长任务协议、Skill 申请协议
- [ ] 约束在文件的开头和结尾都有体现（首尾呼应）
- [ ] 通过 `openclaw gateway restart` 测试 plugin 生效
- [ ] 验证：在对话第 50+ 轮时，约束仍然有效

---

## 附录 C：链接到相关项目

- **Soul Anchor Plugin 完整文档**：`../claude-proxy/soul-anchor.md`
- **Hook 调用 Claude Code 完整代码**：`../claude-proxy/handler.js` 和 `../claude-proxy/README.md`
- **Context Cleaner 方案**：`../claude-proxy/context-cleaner.md`
- **Video Diary 修复和大文件下载方案**：`../claude-proxy/video-diary-fix.md`
- **Skill 申请工作流设计文档**：`../docs/superpowers/specs/2026-03-15-skill-request-workflow-design.md`

---

## 总结

这份指南涵盖了 OpenClaw 多 Agent 系统在生产环境中遇到的 6 个关键问题及其完整解决方案。这些问题看似独立，但共同反映了一个主题：**如何在开放式、长对话的多 Agent 系统中维持硬性约束**。

答案不是单一的技术方案，而是：
- **分层防御**：文件系统 + LLM 约束 + 持续注入
- **流程管控**：工作流 + 人工参与 + Cron 驱动
- **显式可见性**：详细日志 + 告警 + 审计追踪
- **约束韧性**：首尾呼应 + 多次提醒 + 持续验证

希望这份指南能帮助你构建更加可靠、更加可控的 OpenClaw 多 Agent 系统。

**Last Updated: 2026-03-15**
