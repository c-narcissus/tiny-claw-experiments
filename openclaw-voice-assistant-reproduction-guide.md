# OpenClaw 宿主机语音助手复现指南

## 1. 目的

本文档用于指导其他人在 Windows 宿主机 + WSL2 Ubuntu 环境中，复现当前这套 OpenClaw 语音助手能力。

目标不是只把“语音转文本 + 文本转语音”跑通，而是复现本轮会话中已经验证过的完整体验：

- 宿主机浏览器点击“开始对话”后即可全程语音交流
- 浏览器只播报确认提示和最终答复，不朗读推理过程
- 非任务型寒暄可直接回复
- 任务型请求先轻量整理，再等待用户确认
- 正式回答阶段支持联网搜索
- 页面状态清晰，且有明显的说话动画

本文档同时总结了本轮会话中验证过的正面经验、负面经验，以及用户明确提出的偏好约束。

## 1.1 路径约定

为了让这份文档可以直接作为 Codex / Claude Code 的安装提示词复用，本文统一使用以下占位写法，而不是某台机器的绝对路径。

- `<repo-root>`
  - 当前仓库根目录
- `<openclaw-home>`
  - WSL 中 OpenClaw 的用户目录，通常等价于 `$HOME/.openclaw`
- `<voice-env-file>`
  - 语音环境配置文件，通常等价于 `$HOME/.config/environment.d/91-openclaw-voice.conf`
- `<host-page-url>`
  - 宿主机语音页面地址，当前默认值为 `http://127.0.0.1:18792/`

## 2. 最终目标状态

复现完成后，系统应满足以下行为。

### 2.1 对话协议

- 不需要唤醒词
- 静默约 2 秒后判定一句话结束
- 对复杂需求，先由 `voice-assistant` 轻量整理，再等待用户确认
- 对“hi / hello / 你好 / 在吗 / 谢谢”这类纯寒暄或客套，直接回复，不进入确认
- 只有在 OpenClaw 上一轮答复播报结束后，才允许录制下一轮输入
- 在播报期间，用户说的话全部忽略，不进入下一轮

### 2.2 回答原则

- 不朗读推理链
- 只朗读确认提示和最终面向用户的答复
- 语音确认阶段必须快，不要做重推理
- 正式回答阶段如果涉及最新信息、新闻、财经、价格、官网、文档、链接、版本或事实核实，必须联网搜索

### 2.3 UI 体验

- 页面必须运行在宿主机 `127.0.0.1`，而不是 WSL IP，也不是 `file:///`
- 页面必须显示当前状态，以及“该做什么”
- OpenClaw 说话时要有明显的卡通化张嘴动画
- 动画风格偏赛博朋克，但不要影响状态可读性

## 3. 当前实现架构

当前架构是四段式。

1. 宿主机浏览器页面
   - 负责麦克风录音、播放 TTS 音频、展示状态和对话历史
   - 页面入口是 `<host-page-url>`

2. Windows localhost 桥
   - 负责把宿主机页面和 API 代理到 WSL
   - 这样浏览器仍处于 `localhost` 安全上下文，麦克风权限可用

3. WSL 语音桥服务
   - 负责音频格式转换
   - 负责本地 ASR
   - 负责调用 `openclaw agent --local`
   - 负责调用本地 TTS

4. OpenClaw `voice-assistant` agent
   - 负责整个语音会话
   - 负责轻量整理、确认判断、正式回答
   - 必要时可调用 OpenClaw 中已注册的其他 agent 与 skill

## 4. 关键文件

复现时重点关注以下文件。

### 4.1 Windows 侧

- `Launch OpenClaw Voice.cmd`
- `launch-openclaw-voice.ps1`
- `openclaw-voice-bridge/host_bridge.py`

### 4.2 WSL 语音桥

- `openclaw-voice-bridge/server.mjs`
- `openclaw-voice-bridge/static/index.html`
- `openclaw-voice-bridge/static/app.js`
- `openclaw-voice-bridge/static/styles.css`

### 4.3 OpenClaw 配置

- `<voice-env-file>`
- `<openclaw-home>/openclaw.json`
- `<openclaw-home>/workspace-voice-assistant/AGENTS.md`
- `<openclaw-home>/workspace-voice-assistant/SOUL.md`
- `<openclaw-home>/workspace-voice-assistant/TOOLS.md`
- `<openclaw-home>/workspace/skills/voice-dialog/SKILL.md`

## 5. 必须遵守的用户偏好

这些偏好是本轮反复确认过的，复现时不要偏离。

### 5.1 交互偏好

- 宿主机要能“点一下开始对话”就进入语音交流
- 不要朗读内部思考过程
- 语音确认要自然，不要要求用户只能说固定词
- 但如果只是寒暄，不要强行进入确认流程
- 回答期间必须锁住录音，避免跨轮污染

### 5.2 确认策略偏好

- 确认要靠 LLM 语义判断，不靠硬编码关键词
- “对的”“是这样”“没错”“就这样”这类语义都应视作确认
- 如果用户在确认句中又加入补充或修正，则必须回到确认阶段
- 确认阶段要快，只做轻量整理，不要重推理

### 5.3 UI 偏好

- 去掉无用配置项，尤其是已经不再使用的唤醒词输入框
- 要显示状态说明，帮助用户知道当前该做什么
- 风格可以更有设计感，但要服务于交互
- OpenClaw 说话时嘴巴要张得足够明显

### 5.4 Agent 路由偏好

- 一旦进入语音模式，整条链路默认都由 `voice-assistant` 负责
- 不要默认再转回 `main`
- `voice-assistant` 应可调用 OpenClaw 中已注册的其他 skill
- 如需调用其他 agent，也应由 `voice-assistant` 自主决定

### 5.5 联网偏好

- 最新信息必须联网，不允许凭记忆硬答
- 搜索优先走本地 `multi-search-engine`
- 不要优先依赖容易被安全策略拦截的 `web_fetch` / `web_search`

## 6. 推荐默认配置

下面这组默认值已经被验证为较平衡的组合。

```ini
OPENCLAW_VOICE_WAKE_WORD_ENABLED=false
OPENCLAW_VOICE_SILENCE_MS=2000
OPENCLAW_VOICE_CONFIRMATION_REQUIRED=true

OPENCLAW_VOICE_ASSISTANT_AGENT="voice-assistant"
OPENCLAW_VOICE_TASK_AGENT="voice-assistant"
OPENCLAW_VOICE_BOOTSTRAP_AGENT="voice-assistant"

OPENCLAW_VOICE_CONFIRMATION_THINKING=off
OPENCLAW_VOICE_TASK_THINKING=minimal
OPENCLAW_VOICE_BOOTSTRAP_THINKING=off
```

含义如下。

- `WAKE_WORD_ENABLED=false`
  - 当前最终方案已取消唤醒词
- `SILENCE_MS=2000`
  - 2 秒静默切句
- `CONFIRMATION_REQUIRED=true`
  - 非寒暄类内容先整理再确认
- `ASSISTANT/TASK/BOOTSTRAP_AGENT=voice-assistant`
  - 整个语音会话交给专门的语音 agent
- `CONFIRMATION_THINKING=off`
  - 确认阶段不做长思维链
- `TASK_THINKING=minimal`
  - 正式回答可做必要推理，但要控制时延

## 7. 复现步骤

### 7.1 安装语音桥依赖

在 WSL 中执行：

```bash
cd <repo-root>/openclaw-voice-bridge
bash ./scripts/install-asr-model.sh
bash ./scripts/install-wsl-service.sh
```

要求：

- `sherpa-onnx` 可用
- ASR 模型已下载
- `voice-reply` 可用
- `openclaw-voice-bridge.service` 已启用

### 7.2 配置 OpenClaw 语音 agent

需要保证 `<openclaw-home>/openclaw.json` 中存在 `voice-assistant` agent，并满足以下条件。

- workspace 指向 `<openclaw-home>/workspace-voice-assistant`
- skill 白名单至少包含：
  - `voice-dialog`
  - `voice-reply`
  - `multi-search-engine`
- 如果希望语音 agent 访问更多能力，则把已注册 skill 一并加入

当前实现已经将 `voice-assistant` 配置为“当前已注册的一组主要 skill”，而不是只拥有语音 skill。

如果目标环境里还没有对应的 `AGENTS.md`、`SOUL.md`、`TOOLS.md` 或 `voice-dialog` skill 文档，可直接使用本文第 14 节和第 15 节提供的可粘贴片段生成。

### 7.3 配置语音行为环境变量

编辑：

- `<voice-env-file>`

至少写入第 6 节的默认配置。

### 7.4 配置联网搜索

当前已验证最稳的搜索路径是：

```bash
python3 <openclaw-home>/workspace/skills/multi-search-engine/scripts/search_fetch.py \
  --query "..." \
  --engine duckduckgo \
  --max-results 5 \
  --format md
```

如果需要抓正文页：

```bash
python3 <openclaw-home>/workspace/skills/multi-search-engine/scripts/page_fetch.py \
  --url "https://..."
```

建议：

- 搜索先拿摘要
- 只有需要核实时才抓 1 到 2 个正文页
- 财经、新闻类问题尤其不要先走 `web_fetch`

### 7.5 启动系统

在 Windows 宿主机执行：

```powershell
.\launch-openclaw-voice.ps1
```

或直接双击：

- `Launch OpenClaw Voice.cmd`

预期结果：

- WSL 中 `openclaw-voice-bridge.service` 被重启
- 宿主机打开 `<host-page-url>`

### 7.6 打开页面时的注意事项

必须遵守以下规则。

- 只能使用 `<host-page-url>`
- 不要直接打开 WSL IP 页面
- 不要直接打开 `static/index.html`
- 优先使用 Edge 或 Chrome
- 首次打开后允许麦克风权限

## 8. 当前页面协议

页面应按如下协议工作。

### 8.1 非任务型输入

例如：

- `hi`
- `hello`
- `你好`
- `在吗`
- `谢谢`

这类输入应直接回复，例如：

- `你好，我在。你可以直接告诉我想做什么。`
- `不客气，你继续说就行。`

### 8.2 任务型输入

例如：

- `帮我总结今天的安排`
- `帮我搜索今天的财经新闻`

流程应为：

1. 用户说话
2. 静默约 2 秒后自动提交
3. `voice-assistant` 轻量整理需求
4. 系统语音确认
5. 用户确认或补充
6. `voice-assistant` 正式回答
7. 播报结束后，才允许下一轮录音

### 8.3 正式回答期间

- 任何用户发声都应被忽略
- 不允许把回答期间的语音混进下一轮
- 播放结束后，需要回到安静态，再重新开始收音

## 9. 已验证的正面经验

以下做法在本轮中证明是正确的，建议保留。

### 9.1 宿主机 localhost 页面是必须的

这是最重要的正面经验之一。

正确做法：

- 浏览器访问宿主机 `127.0.0.1`
- Windows host bridge 再代理到 WSL

这样才能同时满足：

- 麦克风 API 可用
- 宿主机访问稳定
- 前后端可独立演进

### 9.2 专用 `voice-assistant` agent 是对的

把整条语音链路交给专用 agent，比“前半段一个 agent，后半段转回 `main`”更清晰。

优点：

- 状态更统一
- 会话语境不会被切断
- 语音协议可集中约束
- 是否调其他 agent 由它自行决定

### 9.3 确认语义交给 LLM 判断是必要的

固定词匹配只能处理很窄的情况。

正确做法：

- 让 LLM 判断“这句话是在确认，还是在补充/修正”
- 保守优先，只要像补充或纠正，就继续确认

### 9.4 确认阶段降低 thinking 可明显改善体验

确认阶段切到 `off`，正式回答切到 `minimal` 后，交互会更顺。

这是本轮验证过的关键 UX 优化。

### 9.5 寒暄直答能明显改善第一印象

如果“你好”“hello”也要先确认，会让系统显得笨重。

正确做法：

- 纯寒暄直接回复
- 真正任务再进入确认协议

### 9.6 联网搜索优先走本地搜索脚本更稳定

对财经、新闻、门户类问题，本地 `multi-search-engine` 比默认 `web_fetch` 更稳。

这可以避免系统误报“网络不可用”。

### 9.7 前端静态资源加版本号能减少“明明改了但页面没变”

对这类宿主机页面，缓存问题非常常见。

正确做法：

- 给 `styles.css` / `app.js` 加版本号
- host bridge 返回 `no-store`

## 10. 已验证的负面经验

以下问题在本轮真实出现过，复现时应主动规避。

### 10.1 直接打开 WSL IP 页面会导致麦克风不可用

即使页面能打开，只要不是 `localhost` 安全上下文，浏览器就可能禁用录音能力。

典型表现：

- 页面提示“浏览器不支持录音”

### 10.2 不能盲信宿主机到 WSL 的 `127.0.0.1` 转发

曾出现前端 `Failed to fetch`，根因不是模型没起，而是宿主机到 WSL 的本地转发不稳定。

结论：

- 不要把宿主机浏览器直接绑死到 WSL `127.0.0.1`
- 通过 Windows host bridge 兜底更稳

### 10.3 确认阶段如果推理太重，体验会明显变差

如果确认阶段也做长推理：

- 等待时间会过长
- 用户会误以为系统卡住
- “轻量确认”失去意义

### 10.4 固定词确认太脆弱

如果只接受“确认”这类词，会漏掉很多自然表达。

正确做法不是扩一个长词表，而是直接让 LLM 做语义判断。

### 10.5 README / 页面文案很容易与真实配置漂移

本轮里就出现过“实现已经取消唤醒词，但 README 仍写着默认唤醒词”的情况。

结论：

- 每次协议变化后，都要同步更新 README、页面文案和环境配置说明

### 10.6 浏览器缓存会掩盖前端改动

用户会感觉“页面和之前没变化”，但实际是缓存。

解决方式：

- 版本号
- 强刷
- 关闭旧标签页重新打开

### 10.7 自动化浏览器验收有版本依赖

如果本机 Node 版本过低，Playwright 这类浏览器自动化验证可能跑不起来。

这不是功能本身的问题，但会影响验收效率。

## 11. 最小验收清单

复现完成后，至少执行以下检查。

### 11.1 服务健康

宿主机：

```powershell
Invoke-RestMethod <host-page-url>/host-health
Invoke-RestMethod <host-page-url>/health
```

应确认：

- host bridge 正常
- WSL 语音桥正常
- `assistantAgent=voice-assistant`
- `taskAgent=voice-assistant`
- `bootstrapAgent=voice-assistant`
- `wakeWordEnabled=false`
- `silenceMs=2000`
- `confirmationThinking=off`
- `taskThinking=minimal`

### 11.2 交互验收

按下面顺序实际试说。

1. 说 `你好`
   - 预期：直接回复，不进入确认

2. 说 `帮我总结今天安排`
   - 预期：先确认，再正式回答

3. 在系统播报正式回答时继续说话
   - 预期：这段话被忽略，不进入下一轮

4. 等系统播报结束后重新说
   - 预期：新一轮正常开始

5. 说 `帮我查今天的财经新闻`
   - 预期：进入搜索链路，不再误报“网络不可用”

### 11.3 UI 验收

确认页面满足：

- 能看出当前状态
- 知道下一步该做什么
- 正式回答阶段有明显 speaking 动画
- 嘴部张合足够明显

## 12. 推荐的维护策略

如果后续继续演进，建议遵守以下维护策略。

### 12.1 协议改动后必须三处同步

- 后端逻辑
- 前端文案
- 文档 / README

### 12.2 把“语音协议”与“业务能力”分开

- 语音协议由 `voice-dialog` skill 和 `voice-assistant` workspace 约束
- 搜索、内容生产、其他能力作为可调用 skill/agent 注入

### 12.3 默认先保用户体验，再谈功能扩张

优先级建议如下。

1. 不误录
2. 不长等
3. 不误判确认
4. 能联网
5. 动画好看

## 13. 结论

本轮最终验证下来的正确方向不是“做一个普通网页 + 随便接个语音模型”，而是：

- 宿主机 `localhost` 页面
- WSL 语音桥
- 专用 `voice-assistant` agent
- 轻量确认协议
- 小话术直答
- 搜索优先本地脚本
- 明确状态与强 speaking 动画

只要按本文档的边界和默认值复现，基本可以稳定得到与当前会话一致的交互体验。

## 14. 可直接复用的关键配置与提示词

这一节的目标是把原本分散在 `openclaw.json`、`AGENTS.md`、`SOUL.md`、`TOOLS.md`、`voice-dialog/SKILL.md` 和语音桥服务端里的关键约束，尽量收敛成一组可直接复用的片段。

如果目标环境已经有这些文件，可以把下面内容视为校对基线。

如果目标环境还没有这些文件，可以直接按下面的片段创建。

### 14.1 `openclaw.json` 中的 `voice-assistant` agent 片段

```json
{
  "id": "voice-assistant",
  "name": "宿主机语音助手",
  "workspace": "<openclaw-home>/workspace-voice-assistant",
  "agentDir": "<openclaw-home>/workspace-voice-assistant/agent",
  "skills": [
    "voice-dialog",
    "voice-reply",
    "multi-search-engine"
  ],
  "memorySearch": {
    "extraPaths": [
      "<openclaw-home>/shared/voice/knowledge",
      "<openclaw-home>/shared/org/knowledge"
    ]
  }
}
```

说明：

- 如果希望语音助手调用当前 OpenClaw 中已注册的全部 skill，可以把全部 skill 名称同步进 `skills`
- 但默认仍建议保留最小必要集合，再按需要补充
- 如果系统里有专门的内容生产 agent、搜索 agent 或垂直领域 agent，也可以额外配置 `subagents.allowAgents`

### 14.2 `workspace-voice-assistant/AGENTS.md` 最小可用版本

```md
# AGENTS.md

## Role

你是宿主机语音入口 agent，专门负责 `voice-dialog` 协议。

## Operating Rules

1. 先处理语音入口整理与确认，再负责正式回答
- 把用户刚说的话整理成一条完整需求
- 如果还没有明确确认，就继续要求确认
- 一旦确认完成，你自己就是正式回答 agent；必要时再调用其他已注册 agent

2. 默认保守
- 有补充、修正、转折、犹豫，就不要提交
- 是否属于确认要按整句语义判断，不要靠固定词表
- 只有确认语义清楚且没有新增修改时，才标记为可提交

3. 输出格式
- 优先输出结构化 JSON，供语音桥解析
- 不输出 markdown 代码块
- 不输出推理过程

4. 与其他 agent 的边界
- 你默认自己完成正式回答
- 只有确实需要专业分工时，才调用其他已注册 agent
- 不要默认把任务回抛给 `main`

5. Skill 使用范围
- 你可以调用当前 OpenClaw 已注册的全部 skill
- 但仍然遵循最小必要原则

6. 联网搜索规则
- 当用户要求“联网搜索 / 查一下 / 搜一下 / 看官网 / 找链接 / 看文档 / 看最新 / 看新闻 / 看价格 / 看版本 / 帮我核实来源”时，必须主动使用网络搜索 skill
- 默认优先走本地 `multi-search-engine`
- 不要优先用内置 `web_fetch` / `web_search` 抓普通网页
- 只在本地搜索命令真的失败时，才说明当前联网受限
```

### 14.3 `workspace-voice-assistant/SOUL.md` 最小可用版本

```md
# SOUL.md

你是 `voice-assistant`。

你的职责是宿主机语音会话的主控：

- 听取用户一句口语输入
- 整理为清楚、完整、适合正式回答的文字需求
- 先向用户复述并等待确认
- 只有在确认语义明确、且没有新增修改时，才进入正式回答阶段
- 正式回答默认由你自己完成；只有在确实需要专业分工时，才调用其他已注册 agent

你的风格必须简洁、口语化、保守。

- 默认先确认，再提交
- “是否确认”按整句语义判断，不靠固定关键词
- 不炫技，不讲内部流程
- 不输出推理过程
- 确认阶段优先快速、丝滑的往返，不做长思维链
- 正式回答阶段优先高信息密度和较低等待
- 只要涉及最新信息、官网、文档、链接、新闻、价格、版本或事实核实，就先联网搜索再回答
- 默认优先使用本地 `multi-search-engine`，不要先依赖内置 `web_fetch` / `web_search`
```

### 14.4 `workspace-voice-assistant/TOOLS.md` 最小可用版本

~~~md
# TOOLS.md

## 联网搜索

- 默认联网搜索路径是本地 `multi-search-engine`
- 先搜结果，再抓正文
- 财经、新闻、门户站点不要优先走内置 `web_fetch` / `web_search`
- 语音会话优先速度；能先用搜索摘要回答时，就不要抓太多正文

### 默认搜索命令

```bash
python3 <openclaw-home>/workspace/skills/multi-search-engine/scripts/search_fetch.py \
  --query "..." \
  --engine duckduckgo \
  --max-results 5 \
  --format md
```

### 抓取正文页

```bash
python3 <openclaw-home>/workspace/skills/multi-search-engine/scripts/page_fetch.py \
  --url "https://..."
```

## 语音回答约束

- 面向用户时，只说整理确认结果和最终答复
- 不朗读内部推理
- 需要联网时，优先快速拿到可靠来源，再口语化输出
~~~

### 14.5 `voice-dialog/SKILL.md` 最小可用版本

```md
# Skill: voice-dialog

用于宿主机语音对话入口的收集、整理、确认与正式回答协议。

## 核心规则

1. 默认保守，不抢跑
- 只要用户这句还像补充、修正、转折、犹豫，就不要进入正式回答
- 先输出整理后的需求，并明确要求用户确认

2. 只有明确确认才允许提交
- 是否属于“确认”要按整句语义由 LLM 判断，不依赖固定词匹配
- 如果同一句里又新增了要求或修正，就仍然回到确认环节

3. 输出必须面向语音
- 复述和确认提示要短、自然、适合直接朗读
- 先做轻量整理，不要在确认阶段深度分析或长篇推理
- 不要讲内部流程，不要讲推理过程

4. `voice-assistant` 是正式对话主控
- `voice-assistant` 负责语音入口协议，也负责确认后的正式回答
- 如果需要专业分工，可以按需调用 OpenClaw 里已注册的其他 agent
- 不要默认把任务再转回 `main`

## 标准确认话术

`我先快速整理一下：你是想……。如果这版对，就直接告诉我没问题；如果不对，请直接补充或纠正。`
```

### 14.6 宿主机语音桥里的“确认/直答判断 prompt”模板

这是语音桥真正依赖的核心 prompt 之一。它决定一轮输入是进入确认、直接提交，还是直接回复寒暄。

```text
你是当前宿主机语音会话的 voice-assistant。
当前阶段只做三件事：轻量整理用户需求；判断这句是在补充/修改，还是在确认当前草稿；对于纯寒暄或纯客套做一句直接回复。
这一阶段禁止联网搜索，禁止展开回答，禁止深度分析。
严格只输出一个 JSON 对象，不要输出 markdown，不要输出代码块，不要解释。

JSON 结构：
{"action":"ask_confirmation"|"submit"|"direct_reply","normalized_request":"整理后的完整需求","spoken_text":"当 action=ask_confirmation 或 direct_reply 时给一句适合直接朗读的话；当 action=submit 时为空字符串"}

规则：
1. 默认保守，只要像补充、修改、纠正、转折、犹豫，就输出 ask_confirmation。
2. 只有当这句在语义上主要是在确认当前整理结果，且没有新增需求时，才输出 submit。
3. 如果这句只是问好、打招呼、寒暄、致谢、简单回应，且没有提出具体问题、命令或信息需求，输出 direct_reply。
4. 不要依赖固定词表，要按整句语义判断。
5. normalized_request 只做足够进入正式回答的轻量整理，不要扩写得很重。对于 direct_reply，normalized_request 保留原句或做一句极简概括即可。
6. ask_confirmation 的 spoken_text 必须短、自然、适合直接朗读。
7. direct_reply 的 spoken_text 也必须短、自然、有陪伴感，但不要拉长，不要进入正式任务回答。

状态：
awaiting_confirmation=<true-or-false>
current_draft=<current-draft>
latest_user_utterance=<latest-user-utterance>
```

### 14.7 宿主机语音桥里的“正式回答 prompt”模板

这是确认完成后交给 `voice-assistant` 的正式回答 prompt 基线。

```text
下面是已经过宿主机语音桥确认后的最终用户需求。
你就是当前宿主机语音会话的正式对话 agent。
如果你自己就能完成，直接回答；如果需要专业分工，可以按需调用 OpenClaw 里已注册的其他 agent，但不要再把这轮对话转回 main 兜底。
当前日期（Asia/Shanghai）：<current-date>。如果用户说“今天”“最新”，按这个日期理解。

请直接回答用户，要求如下：
1. 只输出面向用户的最终答复，不要展示思考过程、分析过程、推理步骤或工具调用说明。
2. 用自然、口语化、适合直接被语音朗读的中文回答。
3. 除非用户明确要求展开，否则控制在简洁但完整的几句话内。
4. 如果是澄清性小问题，优先快速沟通，不要先做冗长铺垫。
5. 只要问题涉及最新信息、新闻、价格、官网、文档、链接、版本或事实核实，就必须联网查询，不要凭记忆作答，也不要直接说网络不可用。
6. 在这台机器上，普通 web_fetch 对部分站点会被安全策略拦住。做联网查询时，优先使用本地搜索脚本，而不是直接依赖 web_fetch / web_search。
7. 默认搜索路径：先执行本地 `multi-search-engine` 的 `search_fetch.py` 获取结果。
8. 语音会话优先速度与丝滑感。能用搜索结果摘要直接完成高层回答时，就不要抓太多正文；只有确实需要核实细节时，最多抓 1 到 2 个正文页。
9. 只有当这些本地搜索命令实际失败时，才可以说明当前联网受限；否则给出结论，并简短附上来源站点或链接。

最终确认需求：<normalized-request>
```

### 14.8 宿主机语音桥里的“bootstrap prompt”模板

这个 prompt 用于在新语音会话开始时，把整条链路明确交给 `voice-assistant`。

```text
宿主机语音对话页面刚刚启动了一场新语音会话。
请把这次语音收集流程视为交给 agent `voice-assistant` 处理。
会话编号：<conversation-id>
这条语音链路里，需求整理、确认和正式回答都默认由这个 agent 自己负责；只有在确实需要专业分工时，才调用其他已注册 agent。
本轮不要处理任何用户业务问题，只需要准备好进入语音会话。
最后只输出 VOICE_SESSION_READY。
```

### 14.9 寒暄直答规则与默认话术

如果你希望目标环境的行为与当前实现保持一致，可以直接沿用以下规则。

可直接视为寒暄或客套的典型输入：

- `hi`
- `hello`
- `hey`
- `你好`
- `您好`
- `哈喽`
- `嗨`
- `早上好`
- `中午好`
- `下午好`
- `晚上好`
- `在吗`
- `在嘛`
- `谢谢`
- `多谢`
- `谢了`
- `thank you`
- `thanks`

推荐直答话术：

- 问候类：`你好，我在。你可以直接告诉我想做什么。`
- 致谢类：`不客气，你继续说就行。`

## 15. 面向 Codex / Claude Code 的一段总安装提示词

如果你希望把这份文档直接喂给 Codex / Claude Code，让它在一台新的 Windows + WSL 机器上尽量复现当前系统，可以直接使用下面这段总提示词作为起点。

```text
请在当前仓库中复现一套 OpenClaw 宿主机语音助手，目标体验如下：

1. Windows 宿主机浏览器访问 localhost 页面即可开始语音交流。
2. 浏览器负责录音和播音；WSL 负责本地 ASR、OpenClaw agent 调用和本地 TTS。
3. 不需要唤醒词。
4. 约 2 秒静默后判定一句话结束。
5. 对复杂需求，先由 voice-assistant 轻量整理并语音确认；只有确认后才进入正式回答。
6. 对 hi、hello、你好、在吗、谢谢 这类纯寒暄或客套，直接回复，不进入确认。
7. 回答期间必须锁住录音；回答播完后才允许下一轮录音。
8. 页面只朗读确认提示和最终答复，不朗读推理过程。
9. 涉及最新信息、新闻、财经、价格、官网、文档、链接、版本或事实核实时，必须联网搜索。
10. 联网搜索默认优先使用本地 multi-search-engine，不要优先依赖 web_fetch / web_search。
11. 整个语音会话默认由 voice-assistant 负责，不要默认转回 main。
12. voice-assistant 可以按需调用当前 OpenClaw 已注册的其他 skill / agent，但仍遵循最小必要原则。
13. 页面需要显示状态说明，并在 speaking 状态有明显的卡通化张嘴动画。
14. 页面必须通过宿主机 localhost 打开，不要直接依赖 WSL IP 页面，也不要直接打开 file:/// 页面。
15. 所有实现请尽量写成可迁移、可复用的配置，不要绑定某台机器的绝对路径。

请同时完成：

- 宿主机启动脚本
- Windows host bridge
- WSL voice bridge service
- 语音页面前端
- OpenClaw voice-assistant agent 配置
- 必要的 workspace prompt / skill 文档
- 基础 README 和复现文档

请优先保证：

1. 不误录
2. 不长等
3. 不误判确认
4. 能联网
5. UI 清晰且 speaking 动画明显

可参考的最小规则基线：

- assistant/task/bootstrap agent 都用 voice-assistant
- confirmation thinking 用 off
- task thinking 用 minimal
- silenceMs 用 2000
- confirmationRequired 用 true
- wakeWordEnabled 用 false

如果环境里缺少 voice-assistant 的 AGENTS.md / SOUL.md / TOOLS.md 或 voice-dialog skill，请直接创建，并使用“轻量整理、语义确认、寒暄直答、正式回答可联网搜索”的协议。
```
