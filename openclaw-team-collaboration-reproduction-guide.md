# OpenClaw 团队协作与 3D 可视化复现指南

## 1. 目的

本文档用于指导其他人复现当前这套 OpenClaw 多 agent 团队协作系统，包括：

- 通用团队设计
- workflow 到可视化事件的转换
- mission-control 3D 可视化运行
- Windows + WSL 宿主机访问
- 团队管理者的编排规则
- 真实任务联测与排错

本文档同时总结了本轮会话中验证过的正面经验、负面经验，以及用户明确提出的偏好约束。

## 2. 当前分层

当前体系按三层拆分，必须保持这个边界。

### 2.1 团队设计层

对应 skill：

- `openclaw-team-architect`

职责：

- 定义真实 workflow
- 定义团队角色、边界、依赖、gate、停止条件
- 定义 organizer 是否存在，以及采用哪种 orchestration mode

这一层不能写成“去柜子”“回工位”“画箭头”。

### 2.2 语义适配层

对应 skill：

- `openclaw-team-visual-adapter`

职责：

- 把 workflow 语义转换成 mission-control 命令序列
- 约束 recorded timeline kind、interaction mode、桥接步骤
- 保证语义顺序正确，不让 UI 事件破坏真实依赖关系

这一层负责回答：

- workflow 的“访问共享区域”要如何变成可视化事件
- 什么时候要补 `cabinet-store`
- 什么时候要补 `cabinet-take`
- 什么时候只应该是 `message` / `handoff`

### 2.3 运行时可视化层

对应 skill：

- `openclaw-team-mission-control`

职责：

- 启动浏览器页面
- 记录 timeline
- 驱动 3D 办公室、柜子、箭头、回放、文章页
- 把已转换好的事件播放出来

这一层不应该重新发明 workflow 语义。

## 3. 用户偏好与硬约束

以下偏好已经在本轮系统中反复确认，复现时必须遵守。

### 3.1 关于团队与角色

- 只要是多 agent 团队，就应该能接入同一套通用可视化系统。
- 团队管理者 / 协调者 / organizer 必须真实发挥编排作用，并在可视化里可见。
- roster 必须只包含当前真实参与团队，不能混入 `main` 或其他无关 agent。
- 工位数量必须与可视 agent 数量一致。
- agent 名称必须可见。
- 不同 agent 需要有明显可区分的颜色。

### 3.2 关于 workflow 与可视化

- workflow 编排和可视化编排必须分层。
- workflow 只描述真实协作语义，不描述柜子、工位、桌子、箭头。
- 可视化层可以补桥接步骤，但不能发明假的业务语义。
- 如果 workflow 是 direct edge，就用 `message` / `broadcast` / `handoff`，不要强行画柜子。
- 如果 workflow 是 shared-artifact relay，就必须在可视化里体现 `cabinet-store` / `cabinet-take`。

### 3.3 关于共享资源

- 材料必须先准备好，再通知下游。
- 如果 `agent X` 处理好了 `staff x`，并希望 `agent Y` 基于它继续处理，则顺序必须是：
  - `X work`
  - `X cabinet-store(staff x)`
  - `X 完整返回工位`
  - `message/handoff to Y`
  - `Y cabinet-take(staff x)`
  - `Y 完整返回工位`
  - `Y activate/work`
- 不能出现“先通知、后存放”。
- 不能出现“Y 先取、后收到指令”。
- 如果 gate owner 需要基于共享产物做审核，必须先 `take`，再审核，最后在有新产物时才 `store`。

### 3.4 关于移动规则

- 默认所有 agent 都在自己的工位。
- direct message / reply / handoff 不需要离开工位。
- 只有访问共享柜时才需要移动。
- 任意时刻最多只有一个 agent 移动。
- 下一个移动事件必须等待上一个 agent 完整往返并回到自己工位后才能开始。
- 访问共享柜时，agent 必须足够靠近柜体前脸，不能停得太远。

### 3.5 关于管理轮次

- 团队管理者必须在每个实质性管理轮次结束后判断“继续还是停止”。
- 默认自主上限是 5 个管理轮次。
- 如果超过 10 个管理轮次，必须先问用户。
- 如果要求“至少 3 轮”，指的是至少 3 个实质 workflow round，不是 3 个 UI 事件，也不是 3 个 `discussionRounds`。
- 页面上不能把普通子事件误标成“第 N 轮”。
- 只有 organizer 的结构化 `status` 检查点才是“管理第 N 轮”。
- 最终发布前必须有一次显式“停止”判断。

### 3.6 关于回放与文章页

- 回放必须等待上一个动作完整结束，再播下一个动作。
- 柜子往返不能跳帧。
- 回放要保留完整讨论过程，而不是只保留后几轮。
- `/article` 必须能直接看到最终正文，不允许空白。

### 3.7 关于宿主机访问

- 用户希望最终在宿主机浏览器里看页面。
- 在 Windows + WSL 下，不能盲信 `localhostForwarding=true` 一定有效。
- 必须核对真正监听 `127.0.0.1:3847` 的进程是谁。

## 4. 推荐的通用 orchestration modes

复现时不要把所有团队都硬编码成“协调器统一转发”。

推荐支持的模式：

- organizer-led hub-and-spoke
- workflow-guided peer mesh
- approval chain
- shared-artifact relay
- parallel fan-out / fan-in

选择规则：

- 如果用户沟通、排序、总结都由管理者统一负责，用 `organizer-led`
- 如果专家之间可以直接基于 workflow 协作，用 `peer mesh`
- 如果存在强制审核或发布闸门，用 `approval chain`
- 如果大量依赖共享产物，用 `shared-artifact relay`
- 如果多分支确实独立，再用 `parallel fan-out`

注意：

- 只有真实无依赖分支才能并行。
- 任何 producer-consumer、review、integration 关系都必须保持顺序。

## 5. workflow 到可视化的标准映射

推荐使用 `openclaw-team-visual-adapter` 的 schema 与示例 YAML。

核心映射如下。

| workflow 语义 | mission-control 命令 | timeline kind |
| --- | --- | --- |
| direct notify / reply / handoff | `message` / `broadcast` / `handoff` | `message` / `broadcast` / `handoff` |
| publish shared artifact | `cabinet-store` | `share` |
| retrieve shared artifact | `cabinet-take` | `share` |
| begin local work | `activate` | `work` |
| organizer round decision | `status` | `status` |
| publish final deliverable | `article` | `artifact` |
| mission finish | `complete` | `done` |

### 5.1 顺序模板

#### 有 organizer 路由的共享接力

`producer work -> producer store -> producer return -> organizer route -> consumer take -> consumer return -> consumer work`

#### peer 直连的共享接力

`producer work -> producer store -> producer return -> producer message -> consumer take -> consumer return -> consumer work`

#### 纯共享接力

`producer work -> producer store -> producer return -> consumer take -> consumer return -> consumer work`

#### 审核链

`upstream store -> optional route -> reviewer take -> reviewer work -> reviewer store(review package)`

#### 整合链

`owner store(revised/reviewed package) -> optional route -> integrator take -> integrator work`

## 6. 已验证的正面经验

以下做法在本轮中证明是正确的，建议保留。

### 6.1 三层 skill 分离是对的

- `openclaw-team-architect`
- `openclaw-team-visual-adapter`
- `openclaw-team-mission-control`

这三层拆开后，职责边界更清晰，也更适合跨团队复用。

### 6.2 把 workflow 语义和 UI 隐喻分离是必要的

这是本轮最关键的正面经验之一。

正确做法：

- workflow 写真实依赖和责任
- visual adapter 决定是否补 `cabinet-take`
- mission-control 负责表现

### 6.3 organizer 必须是真实角色

如果团队有 organizer，它不能只是一个名字或只在结尾收尾。

organizer 需要真实承担：

- 任务开始
- 分派下一步
- 轮次判断
- 最终停止判断
- 最终整合与发布

### 6.4 `status` 用作管理轮次检查点是正确的

不要用 `message` 假装“管理判断”。  
显式 `status` 才能让 replay、timeline、审计都保持一致。

### 6.5 共享柜动作串行化是必要的

如果不强制串行：

- 画面会乱
- replay 会跳
- 用户会觉得逻辑不成立

“一个 agent 完整往返后，下一个 agent 才动”是正确约束。

### 6.6 host `127.0.0.1` 不能只看 WSL 配置

正确排查方式已经验证有效：

1. 查 Windows 侧谁在监听端口
2. 查 WSL 侧谁在监听端口
3. 比较两边实际返回的 `/api/state`
4. 如果有 Windows 本地旧进程占着端口，优先替换它

### 6.7 repo / `.codex` / WSL runtime 三处同步很重要

很多“明明改了但没生效”的问题，根因都是三处副本不一致。

## 7. 已验证的负面经验

以下做法已经证明会导致混乱或错误，复现时应明确避免。

### 7.1 不要把 `main` 混进具体业务团队 roster

如果 `main` 被误读进 roster：

- 可视化会出现错误 agent
- 甚至会把 `IDENTITY.md` 一类占位信息当名字读出来

### 7.2 不要让旧页面继续占着目标端口

如果浏览器渲染的是旧 state-file，而 coordinator 写的是新 state-file：

- 页面看起来不动
- timeline 不匹配
- 用户会误判系统失效

### 7.3 不要手改 state JSON 代替真实事件流

这样会造成：

- timeline 不完整
- replay 不可信
- 中间动态过程消失

### 7.4 不要把普通事件误标成 workflow round

`handoff`、`share`、`message` 都可能只是某一轮中的子步骤。  
把它们显示成“第二轮”“第三轮”会误导用户。

### 7.5 不要把审核角色折叠进协调者

如果 workflow 有 dedicated reviewer / audit / QA：

- 这个角色必须真实出现
- 它必须先取件再审核
- 不能由协调器静默代审

### 7.6 不要在 direct edge 上强行移动 agent

direct message / reply 应该留在工位。  
只有访问共享柜才需要离座。

### 7.7 不要假定宿主机一定能访问 WSL 的 `127.0.0.1`

即使 `.wslconfig` 里写了 `localhostForwarding=true`，也可能：

- 宿主机访问不到
- 宿主机看到的是旧内容
- 实际监听者不是想象中的那个服务

## 8. 复现步骤

### 8.1 设计团队

使用 `openclaw-team-architect`：

1. 明确团队目标
2. 选定 orchestration mode
3. 定义 organizer
4. 定义 specialists
5. 定义 shared artifacts
6. 定义 gate owner
7. 定义 stop criteria

输出要求：

- 不包含柜子、桌子、箭头等 UI 术语
- 明确每个角色的真实输入输出

### 8.2 定义 workflow-to-visual 适配

使用 `openclaw-team-visual-adapter`：

1. 给每条 workflow edge 标注语义类别
2. 生成 mission-control 命令序列
3. 校验 timeline kind 是否匹配 runtime 支持集合
4. 检查是否需要补桥接步骤
5. 检查是否错误引入了并行

建议使用：

- schema：`openclaw-team-visual-adapter/schemas/workflow-visual-adaptation.schema.yaml`
- 示例：`openclaw-team-visual-adapter/examples/generic-shared-artifact-review-chain.example.yaml`

### 8.3 准备运行时

使用 `openclaw-team-mission-control`：

1. 准备 roster
2. 启动 mission-control
3. 确认 `/api/state` 不再是 `idle`
4. 确认当前端口上的页面与当前 state-file 是同一个 run

### 8.4 真实任务运行时的检查点

至少检查：

1. roster 是否只包含当前真实团队
2. organizer 是否可见
3. shared relay 顺序是否正确
4. dedicated reviewer 是否真实参与
5. 最终是否有显式停止判断
6. `/article` 是否能直接打开正文

### 8.5 宿主机访问检查

在 Windows + WSL 下：

1. `Get-NetTCPConnection -LocalPort <port>` 看 Windows 侧监听者
2. `wsl ... ss -ltnp` 看 WSL 侧监听者
3. 分别请求 Windows 侧和 WSL 侧的 `/api/state`
4. 如果返回内容不一致，先解决端口占用问题

## 9. 建议的验收清单

复现完成后，至少满足以下条件。

### 9.1 语义正确

- producer 一定先 `store`
- consumer 一定后 `take`
- manager / organizer 的 routing 在正确位置
- reviewer 先 `take` 再 `work`
- integrator 先 `take` 再 `work`

### 9.2 视觉正确

- 工位数 = agent 数
- agent 名称可见
- organizer 可见
- direct edge 不离座
- cabinet 访问才离座
- 每次最多一个 agent 移动
- 柜前停靠点足够接近柜子

### 9.3 回放正确

- 上一个动作结束后才播下一个
- 柜子往返连续
- route line 在回座后消失
- 不跳帧

### 9.4 管理轮次正确

- 只有 `status` 是管理轮次
- 默认不超过 5 轮
- 超过 10 轮先问用户
- 最终发布前有显式停止判断

### 9.5 文章页正确

- `/article` 首屏可见正文
- 标题和内容与当前 run 匹配

## 10. 常见故障与处理

### 10.1 页面不动

常见原因：

- 写事件的 state-file 和浏览器当前页面不是同一个
- coordinator 没有用真实命令流，只是直接改了 JSON

处理：

- 查当前端口实际服务的是哪份 state-file
- 重启或替换旧服务

### 10.2 页面显示了旧任务

常见原因：

- 旧 Windows 本地 mission-control 进程还占着端口

处理：

- 查 `Get-CimInstance Win32_Process`
- 停掉旧进程
- 启动当前 run 对应的新服务

### 10.3 页面里出现 `main` 或奇怪 agent

常见原因：

- roster 生成时用了全量 runtime config
- 没有按当前参与团队过滤

处理：

- roster 只保留当前 team scope

### 10.4 宿主机打不开页面

常见原因：

- WSL localhost forwarding 不稳定
- 宿主机端口被别的进程占用

处理：

- 先查宿主机监听者
- 必要时改用 Windows 本地服务接管目标端口

## 11. 当前推荐文件位置

相关核心文件建议保持如下：

- 团队设计：`openclaw-team-architect`
- 语义适配：`openclaw-team-visual-adapter`
- 运行时可视化：`openclaw-team-mission-control`

不要再把团队设计、语义映射、运行时播放全部混进一个 skill。

## 12. 一句话总结

正确的复现方式不是“先做一个炫酷的 3D 页面”，而是先把真实 workflow 设计正确，再用独立的 visual adapter 把它严格、可审计地转换成 mission-control 事件队列，最后用 mission-control 去播放这套真实而连贯的协作过程。
