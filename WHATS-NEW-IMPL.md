# What's new 里的功能，在产物里都是怎么实现的

对象：`claude_agent_sdk/_bundled/claude` 里那个 **Claude Code 2.1.259**
（版本比官方 digest 最新的 Week 34 / v2.1.239 还新，所以列出的特性理论上都在这个构建里）。
代码树是 `src_full/`，读法见 [`READING.md`](READING.md)。

---

## 怎么读这份文档

**按主题组织，不按周。** 官方的 digest 是"这周发布了什么"，而实现是按模块长的——
比如 auto mode 从 W13 发布到 W32 还在改，把它拆在六个星期里讲，读起来得反复横跳。
所以正文按**实现**归组，每条标上它出自哪一周，文末给出「周 → 条目」的对照表。

每条给四栏：

| 栏 | 含义 |
|---|---|
| **落点** | `文件:行号`，点进去就是实现 |
| **机制** | 数据结构和关键函数，以及触发条件 |
| **证据** | 一到两条原文片段，用来核对我没有编 |
| **在不在产物里** | 最重要的一栏，见下 |

### "在不在产物里"这栏

digest 混了三类东西，它们的"实现"在完全不同的地方：

- **在** —— 逻辑真的在这个二进制里，能读、能改。
- **部分在** —— 客户端这一半在（工具定义、状态机、校验、文案），另一半在服务端或原生模块。产物里读得到的是协议与边界，不是全部。
- **不在** —— 纯 Web / Desktop / 云端侧的功能，产物里只有开关、文案或一个转发调用。

不标清楚的话，很容易花半天找一个根本不在这个二进制里的东西。

### 一个重要前提

产物是**压缩过的**：变量名无意义，但**字符串常量完好**。所以下面所有定位都是靠
UI 文案、报错文案、设置键名、遥测事件名搜出来的，没有一条是靠文件名猜的。
少数有可读文件名的模块（如 `tools__BashTool__UI.js`）是另一次比对的结果，见 `READING.md`。

---

## 一、三张全局表（先当地图用）

### 1. 斜杠命令 → 实现模块

在 `chunk-eawgw36a.js:4163-4241`，是一张惰性加载表。想读某个命令的实现，直接从这儿进：

```js
ultrareview:     () => import("./chunk-ywcyz9et.js"),
"auto-mode-setup": () => import("./chunk-1aw2vrk3.js"),
fork:            () => import("./chunk-fp7f0tvf.js"),
workflows:       () => import("./chunk-jq2gpm60.js"),
```

共 **72 条**。其中 digest 相关的：

| 命令 | 模块 | 命令 | 模块 |
|---|---|---|---|
| `artifacts` | `chunk-tz3c4jvy.js` | `usage` | `chunk-p2grg34j.js` |
| `auto-mode-setup` | `chunk-1aw2vrk3.js` | `theme` | `chunk-gk8dsg45.js` |
| `autofix-pr` | `chunk-jme2g6k2.js` | `ultrareview` | `chunk-ywcyz9et.js` |
| `cd` | `chunk-sb198e74.js` | `fork` | `chunk-fp7f0tvf.js` |
| `config` | `chunk-ycs9nepx.js` | `goal` | `chunk-bem0w31t.js` |
| `design-login` | `chunk-40b7gbfa.js` | `loops` | `chunk-4mdaz05c.js` |
| `effort` | `chunk-t2pwmng1.js` | `remote-control` | `chunk-gn18jndp.js` |
| `fast` | `chunk-sjw5sztx.js` | `remote-env` | `chunk-21ybnb5n.js` |
| `output-style` | `chunk-zx5hvy8r.js` | `workflows` | `chunk-jq2gpm60.js` |
| `powerup` | `chunk-he8nn0kx.js` | `teleport` | `chunk-bfs6prbt.js` |

> 注意 `code-review` **不在这张表里**——它是内置技能，注册在 `chunk-vyy71dp3.js:2449`。
> 命令表、技能表、工具表是三个不同的注册处。

### 2. 工具名总表

在 `chunk-qpwbvc04.js:110345` 起（数组 `REo`）。它比"当前提供哪些工具"更宽——包含
MCP 工具模式、内部工具、权限规则里可出现的名字。digest 里能直接对上的：

| 工具名 | 对应特性 |
|---|---|
| `Monitor` | W15 Monitor 工具 |
| `Workflow` | W22 dynamic workflows |
| `PowerShell` | W13 / W18 Windows 原生 PowerShell |
| `PushNotification` | W16 手机推送 |
| `SendMessage` / `ListPeers` | W32 跨会话消息 |
| `Artifact` / `ClaudeDesign` / `DesignSync` | W25 Artifacts、W34 `/design` |
| `ReportFindings` | W21 `/code-review` |
| `self_hosted_runner_*`（8 个） | W32 self-hosted environments |

### 3. 遥测事件名 = 最全的特性索引

产物里有 **2231 个** `tengu_*` 事件名。想知道某个特性在不在、内部叫什么，
搜事件名往往比搜 UI 文案更快：

```bash
grep -rho '"tengu_[a-z0-9_]*"' src_full/ | sort -u | grep -i <关键词>
```

---

## 二、auto mode：一个反直觉的实现

**先说结论**：auto mode 的"分类器"**不是一个本地小模型**，也不是一个独立的远程服务。
它是**一次普通的 API 调用**（`querySource: "auto_mode"`），提示词是一份
**85 KB 的 Markdown 资产**，运行时把用户配的规则填进占位符再整体发出去。

这条很重要，因为 W19/W25/W28 那些"硬规则"听起来像代码里的判定——
实际上它们是**给模型读的自然语言**。

### 2.1 规则本体：一份资产文件

- **落点**：`assets/permissions_external`（85,239 B，同一份内容还存了
  `assets/permissions_external-0f27b1d1.txt` 和 `assets_named/asset-011.bin`）
- **装载器**：`chunk-zwwbjf7e.js:11`
- **拼装**：`chunk-qpwbvc04.js:211164`（`GTt`）把 `<cc_automode_permissions>` 模板里的
  占位符替换掉；用户规则按 `<user_*_rules_to_replace>` 插入（`chunk-qpwbvc04.js:60567-60613`）

规则分三档，在提示词里就是三个小节：

| 档 | 语义 | 文件位置 |
|---|---|---|
| **HARD BLOCK** | 无条件阻断，不看授权、不看 ALLOW 例外 | `assets/permissions_external:48` |
| **SOFT BLOCK** | 需要用户"点名该操作及目标"才放行 | 各规则条目内 |
| **ALLOW 例外** | 只在它命名的维度上生效（目标/资源/操作） | `:132` |

HARD 那句原文：

```
Block unconditionally. Do not look for authorization. No ALLOW exception applies.
When a HARD rule blocks, the block reason should name the rule and suggest running
the step outside auto mode so the user can review the permission prompt directly.
```

### 2.2 分类器怎么被调用

- **落点**：`chunk-qpwbvc04.js:212506-212540`（两阶段：stage1 快速过滤 + stage2 XML）
- **编排/重试/回退**：`chunk-qpwbvc04.js:212940-213088`
- **默认行为**：**默认 ALLOW**；只有命中 BLOCK 规则才拒绝
- **失败关闭**：分类器不可达或解析失败 → fail-closed（`automode-unavailable` / `automode-parsing-error`）

模型返回的是一段 XML：

```
<block>yes</block><category>Exact BLOCK Rule Name</category><reason>[...] one short sentence</reason>
```

允许时只回 `<block>no</block>`。提示词还规定 `reason` **必须以方括号里的规则名开头**——
这样日志里能直接看出是哪条规则拦的。决策在内部映射成
`{ shouldBlock, reason, category, model, usage, stage, ... }`，
`shouldBlock:true` → 权限行为 `deny`，`denialKind: "automode-blocked"`。

### 2.3 用户规则与信任边界

设置项 `autoMode` 的 schema（`chunk-mzmfq60a.js:5381-5411`）：

```js
autoMode: {
  allow:         string[],   // 元素可含字面量 "$defaults" = 在此位置继承内置规则
  soft_deny:     string[],
  hard_deny:     string[],
  environment:   string[],
  classifyAllShell: boolean, // true 时挂起所有 Bash/PowerShell 的 allow 规则，全部走分类器
}
```

**信任边界**（`chunk-67nd7etf.js:3573-3588`）：只有 `userSettings` / `flagSettings` /
`policySettings` 能设分类器规则。`projectSettings` 和 `localSettings` 里的 `autoMode`
会被**忽略并告警**，理由是这两个来源"仓库可控（repo-controllable）"——

```
settings autoMode in ${A} ignored — only user/flag/managed settings may set classifier rules
```

这是一条挺漂亮的供应链防线：克隆一个仓库不该能改你 agent 的安全策略。

### 2.4 各周特性分别落在哪

| 周 | 特性 | 在规则文件里的位置 |
|---|---|---|
| W19 | hard deny 不看 allow 例外 | `:48` HARD BLOCK 段 |
| W25 | 阻断破坏性 git 命令 | `:59` `Git Destructive`（force push / 删远程分支标签 / 改写远程历史 / 非本会话的 `--amend`）；`:82` `Irreversible Local Destruction`（`reset --hard`、`clean -fd`、`stash drop`） |
| W28 | 阻断 transcript 篡改 | `:113` `Session Transcript Tampering`——拦对 `~/.claude/projects/**.jsonl` 的写入，以及伪造分类器读的 `{"meta":…}` 真值行；**读**不拦 |
| W28 | 未解析变量的 `rm -rf` 先问 | `:83` `Unverifiable Deletion Target`——`$VAR` 无法从本窗口推导时 **fail closed**。清除方式是**把解析后的字面路径写进命令重跑**；`echo "$VAR"` 不算，因为它的输出到不了分类器 |

规则枚举 ID（`git_destructive`、`session_transcript_tampering` 等约 60 项）在
`chunk-qpwbvc04.js:60620`（数组 `X3r`），可用于对日志归类。

### 2.5 平台与计划的可用性

- **落点**：模型门控 `zie(model)` `chunk-x722nt0q.js:16162-16182`；
  计划级断路器 `MQe()` `chunk-x722nt0q.js:15835`；总门 `chunk-qpwbvc04.js:139069`
- **机制**：可用 = 总门通过 **且** 模型支持 `supportsAutoMode` **且** 断路器未跳。
  断路器由服务端 GrowthBook 标志 `tengu_auto_mode_config.enabled` 驱动——
  **"Pro 计划可用"不是硬编码字符串，是服务端下发的开关**。
- 非第一方且非 `qP`（AWS/GCP 托管）的厂商上，最新模型返回 false——所以
  Bedrock / Vertex / Foundry 的可用性确实与第一方不同（这与官方说明一致）。

### 2.6 `/auto-mode-setup` 与 CLI 子命令

- **向导**：`chunk-1aw2vrk3.js`（`"Teach auto mode about your environment?"` 在 `:374`）
- **提案**：`chunk-pysvwae3.js`（`querySource: "auto_mode_setup_propose"`）
- **落盘**：`chunk-5ms2zeb7.js:2467-2596`，返回 `autoModeKeysWritten`（实际写入了哪几个子键）
- **流程**：问 posture（Work / Open source / Hobby / Mixed）→ 是否额外扫 shell 历史和其他仓库
  → 扫本仓库 + 近期会话 → 调模型产出 `{environment, allow, soft_deny, hard_deny, notes}` 提案
  → 用户确认后写进 `userSettings.autoMode`，并可顺手从 `permissions.allow` 里移除被标记的危险项
- 另有 CLI 子命令 `claude auto-mode {defaults,config,reset,critique}`（`chunk-f7m7ykk9.js:5927`）

---

## 三、权限、审查与安全

### 3.1 `/code-review`（W21 起，W30 改成后台子代理）

- **落点**：技能注册 `chunk-vyy71dp3.js:2449`；fork 执行 `chunk-6wna9z6m.js:430-484`
- **机制**：它是**内置技能**不是内联提示词，用 `no({name, aliases:["review"], ...})` 注册。
  W30 的"改用后台子代理"具体就是 `getContext` 的默认返回值变了：

```js
getContext(e, o) { if (xs()) return "inline"; if (so(o)) return "inline"; return "fork"; }
```

  （`chunk-vyy71dp3.js:2478-2480`——只有两种情况退回 inline，其余一律 `"fork"`。）
  带 `context:"fork"` 的技能会先冻结自己的 allowedTools/denies，再作为子代理启动。
  低/中 effort 档在这个上下文里**自己**跑 8–10 个 finder 角度（提示词明确写
  "do NOT spawn subagents"），高档才再 fan-out 出去。
- **在产物里**：在。

### 3.2 `/ultrareview`（W17）与 `claude ultrareview`（W18）

- **落点**：命令 `chunk-ywcyz9et.js`；提交/回传 `chunk-t73wj8h5.js:137,158,788,815`；
  CI 子命令 handler `chunk-hwbeehqn.js:246-489`
- **机制**：flag 解析认 `--fix` / `--comment` / `--post|--no-post`，其余当 target。
  `/code-review ultra` 与 `/ultrareview` 同源——code-review 技能注册时写了
  `subcommands: { ultra: "ultrareview" }`。
  提交时创建云端会话：固定环境 `env_011111111111111111111113`、`tags:["ultrareview"]`，
  用一组 `BUGHUNTER_*` 环境变量驱动多 agent 舰队
  （`BUGHUNTER_FLEET_SIZE` 默认 5、上限 20；还有 `BUGHUNTER_DRY_RUN`、`BUGHUNTER_MAX_DURATION`）。
  PR 模式取 `refs/pull/<n>/head`，分支模式走 git bundle。
  CI 子命令则 `singlePass:true, skipTaskRegistration:true`，用 SSE 轮询进度
  （`E3(sessionId, lastEventId)`，3 秒一次，断连重试 5 次），从进度块解析
  `{bugs_found, bugs_verified, bugs_refuted}` 打到 stderr。
- **在产物里**：在（提交与回传逻辑完整；实际舰队跑在云端）。

### 3.3 `/autofix-pr`（W13 / W15）

- **落点**：`chunk-jme2g6k2.js`（整文件）
- **机制**：`remoteTaskType:"autofix-pr"`。先用 `gh pr view` 探测当前分支的开放 PR
  （拒绝默认分支/已合并/已关闭），并查 taskRegistry 避免重复提交。
  然后 `iT({source:"autofix_pr", githubPr:{...}, useDefaultEnvironment:true})` 建云会话，
  订阅 PR webhook 当推进来源；**若 Remote Control 没连或 app 没装，回落到 30 分钟一次
  的轮询 cron 兜底**（`rke("*/30 * * * *", …)`）。
- **在产物里**：在。

### 3.4 五类云端任务共用一套骨架

`chunk-qpwbvc04.js:111404` 的白名单说明了产物里认识的全部云端任务类型：

```js
var cTo = ["remote-agent", "ultraplan", "ultrareview", "autofix-pr", "remote-workflow"];
```

它们共用同一个云会话创建器 `iT(...)` 和任务登记器 `Wde(...)`，
差异只在 `source` / `remoteTaskType` / 注入的环境变量。

---

## 四、Artifacts 与 `/design`

### 4.1 Artifacts 本体（W25）

- **落点**：`chunk-8v4jrmvb.js`（整个 Artifact MCP 工具；动作路由 `:1462`，
  发布 schema `:3853-3911`，发布落地 `:16297-16360`）；
  工具描述与页面契约 `chunk-5vsy4tn2.js:113-274`；prompt 拼装 `chunk-3enj1an5.js:80-183`
- **机制**：单个 MCP 工具，用 `action` 分派：`publish`（默认）/ `read` / `list` /
  `delete` / `open` / `pin` / `upload_asset` / `list_files` / `read_file`。
  发布参数 `file_path` + 首发必填 `favicon`（emoji）+ 可选 `description`；
  带 `url` 就是原地更新同一个 artifact。
- **"页面能发布新版本"** 由 **artifact-publish capability** 表达——页面声明
  `capabilities`，就获得了保存自身新版本的能力。并发是**整文档 compare-and-set**：
  冲突返回 `conflict` 并把新版交回让你合并，只有显式 `force:true` 才丢弃。
- **在产物里**：在（客户端工具、schema、版本冲突逻辑齐全；托管在 claude.ai）。

### 4.2 Artifacts 调 MCP connectors（W29）

- **落点**：capability 家族校验 `chunk-8v4jrmvb.js:11822`；发布预览的连接授权标注 `:15147`；
  capabilities roster 加载器 `chunk-vyy71dp3.js:618-663`
- **机制**：访客侧的 connector 通过发布时的 `capabilities.mcp` manifest 表达。
  发布结果会标注页面是否 "carries a stored connector grant" 或 "clears stored connector grant"。
  `artifact-capabilities` 技能加载时会把用户 roster 里的 mcp host servers 解析出来注入会话。
- **在产物里**：**部分在**——授权声明、manifest 校验、host-server 解析在这边；
  "访客用自己的 connector 执行"在服务端运行时。

### 4.3 分享、角色与来源

- **落点**：audience 枚举 `chunk-tz3c4jvy.js:379`（`["owner","users","org","public"]`）；
  audience 文案 `chunk-8v4jrmvb.js:12752-12764`；只读拒绝码在页面载荷里
  （`assets_named/asset-045.bin` 的 `READER_CODES`）
- **机制**：可见性用 `audience` 表达（`public` 就是公开分享链接）；
  无写权限时页面载荷用 `not_writer` / `not_granted` / `not_declared` 家族把视图降级为只读；
  Claude Tag / agent 创建的 artifact 用 `agent_scoped` 与画廊的 `source_surface` 字段标记来源。
- **在产物里**：**部分在**——开关、文案、只读降级在这边；写权限授予与公开链接生成在服务端。

### 4.4 `/design`（W34 artboard 研究预览）

- **落点**：命令 `chunk-9zd649cc.js:243-281`（`isEnabled: vAn() && VE()`）；
  技能正文 `assets_named/design.md`；文件包 `chunk-401akmyd.js`；
  播种器 `assets/seed-canvas.mjs`；编辑器载荷 `assets_named/payload.template.html.asset`（约 2 MiB）
- **机制**：`/design <brief>` 让 Claude 把设计写成 `.dc.html` 工作文件（每个 = 一个 artboard，
  各自独立 sandbox iframe），加一份 `canvas.json` 布局清单，用 `seed-canvas.mjs`
  把它们播种进 payload 副本，再用 `Artifact` 工具发布（固定 `contract:"0.1.31"`）。
  能否 Save 取决于账号是否有 artifact-publish capability，否则是只读 + PNG/PDF 导出。
- **artboard 的数据结构**在 `assets/seed-canvas.mjs:93-94`：

```js
const CANVAS_KEYS   = ['artboards', 'annotations', 'launch', 'pages']
const ARTBOARD_KEYS = ['file','x','y','w','h','title','expand','print','page','is_interactive']
```

- **与 `design-login` 的关系**：canvas 技能把账号类动词转发出去——
  `subcommands: {sync: "design-sync", login: "design-login", consent: …, revoke: …}`，
  且 `subcommandsBareOnly: true`，所以 `/design login` 走的是独立的 `design-login` 命令。
- **两个 `/design`**：产物里有一个更早的 Claude Design 中枢（`chunk-mcc10fjt.js:101`，
  `isEnabled: bSe()`，转发到 `get_claude_design_prompt` 等原生工具），与上面这个
  artboard 画布技能**并存且互斥**。读的时候别搞混。
- **在产物里**：在（命令、技能文本、播种器、2 MiB 编辑器载荷全部打包；账号后端不在）。

### 4.5 内置技能是怎么注册的

- **落点**：注册器 `no({...})`；`artifact-design` `chunk-vyy71dp3.js:733`、
  `artifact-diagramming` `:748`、`artifact-capabilities` `:596`；
  名称常量 `chunk-fxf5eq5t.js:10-12`；bundled-skill 开关
  `CLAUDE_CODE_DISABLE_BUNDLED_SKILLS`（`chunk-f6ht09n5.js:13`）
- **frontmatter 契约**：`name / description / whenToUse / isEnabled / userInvocable / menuDescription`
- **三种加载形态**：
  - `artifact-design`：`userInvocable: false`——只由模型在写 artifact 前自行加载
  - `artifact-diagramming`：`userInvocable: true`——用户也能调
  - `artifact-capabilities`：**动态技能**，没有静态 md 正文。每次加载都用凭据
    实时抓取用户 roster，现组合出可用 capability 列表、`.d.ts` 类型定义和 mcp host servers
- **在产物里**：在（注册器、契约、design/diagramming 正文打包；capabilities 名单按用户下发）。

---

## 五、子代理、fork 与 dynamic workflows

### 5.1 fork 是什么，和普通子代理差在哪（W33）

- **落点**：门控 `chunk-qpwbvc04.js:142194-142222`；差异分支 `:143336-143350`
- **机制**：fork 是一个内置 agent 类型（`model:"inherit"`、`permissionMode:"bubble"`、
  `tools:["*"]`），**只在显式传 `subagent_type:"fork"` 时用，永远不是默认**。
  真正的差别在 `Task.call` 里：fork 分支会带上

```js
override.replHydration = { kind: "fork", log: replayLog }
forkContextMessages: ze ? v.messages : void 0
```

  也就是把**当前会话的全部 messages** 加上 REPL 的 replay 日志复制过去；
  普通子代理只有自己的 systemPrompt 和受限工具集。
- **默认开**：门控 `OHr()` 在未被 `CLAUDE_CODE_FORK_SUBAGENT=false`、受限模式或
  GrowthBook `tengu_fork_subagent_enabled` 关掉时返回 `"default"`。

### 5.2 `/fork` 命令（W29）

- **落点**：`chunk-fp7f0tvf.js:323-434`
- **机制**：用 `sle(messages, …)` 从当前对话构造 seed，再
  `txe(..., "fork_session", …, { keepParent: true, storageV5 })` 把整段对话拷进一个
  **新的独立后台会话**。`keepParent:true` 保证当前会话继续可用；新会话作为独立 session
  出现在 roster 里，可以用 SendMessage 互发消息。埋点 `tengu_session_fork`。

### 5.3 子代理嵌套深度（W24）

- **落点**：`chunk-y91b9t7k.js:11-27`（上限）、`chunk-x722nt0q.js:24754`（深度计数）、
  `chunk-qpwbvc04.js:142837`（判定）
- **机制**：深度 = `agentType === "main" ? 0 : (depth ?? 0)`，新子代理 `+1`。
  超过上限抛错并提示用户抬高环境变量。

  **这里和 digest 对不上**：digest 说"后台链最深 5 层"，而产物里的默认常量是

```js
var o = 3, _ = "tengu_hazel_trellis";
```

  **3**，且可被 `CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH` 或 GrowthBook
  `tengu_hazel_trellis` 覆盖。产物里搜不到写死的 5——那个 5 是服务端下发的值。
  另外还有一条独立的 observer 扇出上限 `igo = 2`（`chunk-qpwbvc04.js:102681`）。

### 5.4 子代理默认后台（W27）与权限弹窗回传（W26）

- **落点**：默认判定 `chunk-qpwbvc04.js:143073`；canUseTool 传递 `:143300-143327`
- **默认后台**：`(!Oe && d !== !1)` 这一项——只要不是 teammate、又没**显式**传
  `run_in_background:false`，就是后台。fork 门控开着时输入 schema 干脆把这个参数
  `omit` 掉，子代理一律后台。
- **弹窗回传**：spawn 时无论前台后台都把**父会话的** `canUseTool` 和 `toolUseContext`
  交给子代理，所以后台子代理要权限时，请求经主会话上浮，在 agents 视图里作为
  待批项出现（`worker_status === "requires_action"` → 渲染 `awaiting input`）。

### 5.5 dynamic workflows（W22）

- **落点**：harness `chunk-d361k418.js`（沙箱 `2864-2938`；原语 `2500-2848`；上限 `1264-1327`）；
  开关 `chunk-7kkptxyp.js`
- **脚本 API**：注入 VM 全局的 `agent(prompt, opts)`、`parallel([() => agent(), ...])`、
  `pipeline(items, stage1, stage2, ...)`，加上 `phase`、`log`、`console`、`budget`、
  `setTimeout`、`args`。
- **沙箱**：Node `vm` 的 `createContext(..., { codeGeneration: { strings: false, wasm: false } })`
  ——把 `eval` / `Function` 关掉。
- **约束**（这些数字是硬编码的，值得记）：

| 约束 | 值 | 位置 |
|---|---|---|
| `agent()` 总调用上限 | `1000`（超出抛 `WorkflowAgentCapError`） | `chunk-d361k418.js` |
| 本地并发 | `min(16, max(2, cores-2))` | `:1262` |
| 远程并发 | `50` | — |
| 嵌套 | **只允许一层**，子 workflow 内再调 `workflow()` 直接 reject | `:914` |
| 单 agent 超时 | `180000` ms | — |
| token 预算 | 超了抛 `WorkflowBudgetExceededError`（in-flight 保留） | — |

- **在产物里**：在。本 CLI 自己就带 workflow 工具和 `workflow-subagent` 子代理类型。

### 5.6 Agent view（W20）与状态词（W28）

- **落点**：CLI 入口 `chunk-828gjfwt.js:226-301`；视图与分类 `chunk-nc12g7wp.js:700-741`；
  标题生成 `chunk-5xpbtbhh.js:19-45`
- **状态词**（`chunk-nc12g7wp.js:708-711`）：

```js
{ review: "Ready for review", blocked: "Needs input", working: "Working", done: "Completed" }
```

  分类依据 `worker_status`：`busy`→working、`waiting`→blocked、有未批子项→review、
  `success/failure/stopped`→done。
- **"分类器写的标题"** 确实是一次独立的模型调用：prompt 是
  `Generate a short kebab-case name (2-4 words) ...`，这次调用的 `canUseTool` 直接 deny
  （"Session name generation cannot use tools"）、`maxTurns: 1`，产出的 `fix-login-bug`
  之类就是会话名。

---

### 5.7 后台会话进 `/resume`、被"pin 住"不退出（W21）

- **落点**：`/resume` 命令 `chunk-eawgw36a.js:4210`；保活 `chunk-qpwbvc04.js:143392-143398`；
  daemon 空闲退出条件 `chunk-vg618r4r.js:3273-3277`；保活周期 `chunk-345rc3pn.js:21,44`
- **机制**：后台会话由 daemon 托管，按 `session_keepalive_interval_v2_ms`（默认 **120000** ms）
  保活；`claude agents` 和 `claude --resume` 都会列出后台会话。
  当有 REPL 作为 owner attach 时，会话进入

```
Agent worktree kept at ${ks}: backgrounded owner awaits keepalive, resume pending
```

  这个状态而**不被回收**。daemon 只在"没有 attach 的 `claude agents`、且没有存活 worker"时
  才空闲退出——也就是被占用着就不退。
- ⚠️ 产物里**没有一个显式叫 "pin" 的实现**：`pinSessionId`（`chunk-qpwbvc04.js:238372`）
  只是个恒等透传函数，与"pin 住不退出"无关。digest 说的 "pin" 在代码里等价于上面这套
  owner / keepalive 占用机制。

---

## 六、模型、effort 与快模式

### 6.1 产物认识哪些模型

- **落点**：别名常量 `chunk-40r0jxw7.js:136-154`；能力目录 `chunk-xfn8hpdj.js:270-660`
- **id 常量**（原文）：

```js
FABLE_ID: "claude-fable-5-1",   MYTHOS_ID: "claude-mythos-5-1",
OPUS_ID:  "claude-opus-5",      PREV_OPUS_ID: "claude-opus-4-8",
SONNET_ID: "claude-sonnet-5",   PREV_SONNET_ID: "claude-sonnet-4-6",
HAIKU_ID: "claude-haiku-4-5",
```

  每个 id 挂着 `family / context / capabilities / default_effort / effort_cost_index / provider_ids`。
  选择器主档是 Opus 5 / Sonnet 5 / Haiku 4.5，4.8 及更早标为 Legacy。

### 6.2 effort 档位

- **落点**：`chunk-t2pwmng1.js`（整个 `/effort` 命令）
- **五档 + 一**（`:417-423`）：

```js
{ value: "low", color: "warning" }, { value: "medium", color: "success" },
{ value: "high", color: "permission" }, { value: "xhigh", color: "autoAccept-shimmer" },
{ value: "max", color: "rainbow-animated" },
```

  当模型支持 workflows 时追加第六档 **`ultracode`**（子标签 "xhigh + workflows"）。
- **默认值**：ultracode 资格 → 组织覆盖 → 会话值 → 模型目录里的 `default_effort`
  （Sonnet 5 / Opus 5 都是 `"high"`），兜底索引 `Ao = 3`。
- 滑块用 ←/→ 调档，`s` 只对本次会话生效。

### 6.3 fast mode（W16 / W20 / W22 / W30）

- **落点**：`chunk-eawgw36a.js:3254`（选择器）、`:3635`（`/fast` 入口）、`:3320-3520`（切换与计费文案）；
  命令壳 `chunk-sjw5sztx.js` 只是 re-export
- **机制**：一个布尔应用状态 `fastMode`，标注 research preview，有 cooldown 状态。
  绑定到具体模型（兜底 `claude-opus-5`），**切到别的模型会自动关掉**。
  开启前先跑 PreModelSwitch 检查，失败回滚。遥测 `tengu_fast_mode_toggled`。

### 6.4 adaptive thinking 默认开（W27）

- **落点**：`chunk-tdyswf43.js:97`、`chunk-x722nt0q.js:15967-15993`
- **机制**：请求构造时没有显式 thinking，就按全局开关取：

```js
thinkingConfig: S ?? (sN() !== false ? { type: "adaptive" } : { type: "disabled" })
```

  `sN()` 除非 `MAX_THINKING_TOKENS=0` 或 `alwaysThinkingEnabled === false` 否则为真；
  模型是否支持由能力表的 `adaptive_thinking` 决定（Sonnet 5 支持，4.5/4.1 显式排除）。

### 6.5 1M 窗口与 prompt cache（`/cd` 为什么不重建缓存）

- **落点**：窗口常量 `chunk-xfn8hpdj.js`（`context.window: 1e6, native_1m: true`）；
  缓存断点构造 `chunk-qpwbvc04.js:205956`；TTL 策略 `Prr()` `:205959`；失效判定 `:130209`
- **机制**：缓存断点写成 `{ type: "ephemeral", ttl?, scope? }`，`scope` 只在 `"global"`
  时带上（否则是会话级），`ttl` 取 `5m`（默认）或 `1h`（订阅者，由
  `tengu_prompt_cache_1h_config` allowlist 决定）。
  **缓存键的实际构成是"到断点为止的内容前缀 + scope + ttl"——不含 cwd 或路径分量。**
  失效判定只有一句 `cache_control changed (scope or TTL)` 算变化。
  所以 `/cd` 换工作目录不会改动被缓存的前缀，自然不触发重建。
- `/cd` 本体的实现是另一位调查员的范围（`chunk-sb198e74.js`）。

---

## 七、会话与命令体验

### 7.1 `/goal`（W20）

- **落点**：命令 UI `chunk-bem0w31t.js:321`；引擎 `chunk-xz3v8y7y.js`
- **机制**：`/goal <condition>` 把那句自然语言条件注册成一个**会话级 Stop 钩子**：

```js
o.sessionHooksRegistry.add(r, "Stop", "", { type: "prompt", prompt: t })
```

  每当模型想停下来，这个 prompt 型钩子评估条件；不成立就阻止停止、继续干；
  成立后自动追加 `goal_status{met:true}` 附件并清除自己。上限 4000 字符。

### 7.2 `/loop` 的自调速（W15 / W33）

- **落点**：技能 `chunk-xv4q8hdd.js`；调度工具提示 `chunk-96acb4pv.js:13`；
  管理 UI `/loops` `chunk-4mdaz05c.js`
- **机制**：`/loop [interval] <prompt>`。**带间隔** → 用 schedule 工具建 cron
  （recurring，7 天后自动过期）。**不带间隔** → 动态模式，也就是"自调速"：
  本轮照常干活，然后把决定权交给模型——在本轮最后一个动作里调用 schedule 工具，
  自己选 `delaySeconds`（若同时 arm 了 Monitor，就把 1200–1800 秒当兜底心跳）、
  给出 `reason`，并把原样带 `/loop ` 前缀的 prompt 传回来以便下次重入技能；
  不需要继续了就 `stop: true`。
  所以"自调速"**不是产物里的调度算法，是技能提示词指导模型自己定**。
- `/loops` 命令管理的是"本会话生效的 cron 与 stop-hook"，内部复用 `/goal` 的机制。

### 7.3 session recap（W17）

- **落点**：生成 `chunk-hpwwgy4t.js`；状态机 `chunk-eawgw36a.js:78330-78440`；
  设置键 `awaySummaryEnabled`（`chunk-mzmfq60a.js:6909`）
- **机制**：离开 **5 分钟以上**回来触发一次性 recap。用一个很小的请求生成
  ≤40 词的纯文本摘要，提示词是 "The user stepped away and is coming back.
  Lead with the overall goal and current task, then the one next action"。
  开关优先看 `CLAUDE_CODE_ENABLE_AWAY_SUMMARY`，其次 `awaySummaryEnabled !== false`。
  已有新回合在跑就丢弃。

### 7.4 自定义主题（W17）

- **落点**：`chunk-gk8dsg45.js`（`/theme`）；插件主题 `chunk-eawgw36a.js:16571`；
  manifest 的 `themes` 字段 `chunk-mzmfq60a.js:4401-4408`
- **机制**：可以从预设 fork 或新建配色，存成主题 JSON（用 `setPreviewOverrides`
  实时预览）。插件通过 `themes/` 目录或 manifest 声明发布主题，运行时并入 `customThemes`。
  **safe mode 下自定义主题被禁用。**

### 7.5 屏幕阅读器模式（W29）

- **落点**：Ink 渲染器 `chunk-2kd50vvr.js:16545-16776`、`:17810`；
  Markdown 渲染分支 `chunk-b12pmjnq.js:642,654`
- **机制**：由 `CLAUDE_CODE_ACCESSIBILITY`（→ `accessibilityMode`）和
  `INK_SCREEN_READER`（→ `isScreenReaderEnabled`，需 TTY）驱动。
  开启后渲染器改走线性路径：不发隐藏光标序列、不做全屏重绘，
  Markdown 等可视组件改以平铺文本输出。影响的是整个 Ink 输出层。

### 7.6 `/usage` 按来源拆解（W16 / W21）

- **落点**：命令壳 `chunk-p2grg34j.js:344`；聚合与渲染 `chunk-17cqagxf.js:2947-3157`
- **机制**：**扫描本地会话记录**按天/周聚合，产出
  `totalCost / requestCount / sessionCount` 加分类数组
  `skills / agents / plugins / mcpServers / loops / behaviors`。
  归类键：插件用 `plugin:${name}@${marketplace}`、MCP 用 `server:${name}`、
  subagent 与 skill 按名字。渲染成 Skills / Subagents / Plugins / MCP servers 四个分区。

### 7.7 `/radio`（W27）

- **落点**：`chunk-99syxh2g.js:35`，命令元数据 `chunk-qpwbvc04.js:216301`
- **机制**：就是打开一个网页：

```js
if (await Xr("https://clau.de/radio"))
  return { type: "text", value: "Opening Claude FM in your browser…" };
```

  音频不在产物里。注意 `/radio` 不在上面那张 `Xse` 命令表内，走的是另一处注册。

### 7.8 `/powerup`（W14）

- **落点**：`chunk-a9v3xx4g.js:295`（课程数组 `aw`）、`:838-926`（解锁进度）
- **机制**：**课程内容硬编码在代码里**，每课 `{id, title, tagline, body}`，
  `body` 就是一个 Ink/React 组件（带动画帧）。十课：
  `at-mentions / modes / undo / background / memory / mcp / automate / subagents /
  cross-device / model-dial`。解锁状态存 storage 的 `powerupsUnlocked`。

### 7.9 worktree 与 GitLab（W19 / W33）

- **`worktree.baseRef`**：值域 `"fresh"`（默认）/ `"head"`，schema 在
  `chunk-mzmfq60a.js:6493`，消费点 `chunk-qpwbvc04.js:126239`
  （`worktree?.baseRef === "head"` → `fromHead`）。`fresh` 从
  `origin/<默认分支>` 拉，`head` 从当前本地 HEAD 拉。
- **GitLab 支持**：托管方归一化 `chunk-f2q86n8r.js:21-30`
  （`altssh.gitlab.com → gitlab`），MR URL 识别 `chunk-1mxa8xnf.js:253`
  （`/-/merge_requests/<n>` 与 GitHub 的 `pull` 一起归为 `"pr"` 类型）。
  marketplace 的允许 host 集合里含裸 `gitlab.com`，可以直接克隆。

### 7.10 `/team-onboarding`（W15）

- **落点**：分享工具 `chunk-7g99bwvs.js`（`ShareOnboardingGuideTool`），
  命令侧扫描在 `chunk-g7wzg5br.js` / `chunk-nv0q3ger.js`
- **机制**：扫描本地会话数据生成一份 guide 文件（**模型不能调用它**，
  提示词里明确写 "it scans local session data and cannot be invoked by the model"），
  再由工具用 `check/create/update/delete` 四种模式上传，POST 到 org 端点拿到
  `share_url` + `short_code`。产出是一个可分享、可重放的 org 级 onboarding 链接。

---

### 7.11 `/cd`（W24）

- **落点**：命令入口 `chunk-sb198e74.js:660`；重定位核心 `chunk-b5hwpr6n.js:219-263`
- **机制**：这里有个容易想反的地方——**不是"缓存键里排除了 cwd"，而是"cwd 从来没进过会变的部分"**。
  cwd 烘焙在会话开头那段静态的 *environment block* 里，而那段属于被缓存的 system prompt，
  `/cd` 从不重写它，所以缓存键不变、缓存继续命中。
  换了目录之后，它改为往对话里**注入一条消息**告诉模型：

```
The session's working directory has changed to ${g} (…via /cd…).
The environment block at the start of this conversation still names the previous
directory — that information is stale.
```

  同时后台做重定位：刷新 settings / hooks / skills / plugins / git branch，`invalidateAllRenders()`，
  `refreshConfig()`。

### 7.12 `/config key=value`（W25）

- **落点**：入口 `chunk-ycs9nepx.js:346`；解析与写入 `chunk-92xc0hye.js:32`（`WVe` 解析）、`:82`（单键写入）
- **机制**：**schema 驱动**。设置清单由 `Tbe()` 得到，按 `id` 匹配到一条 setting descriptor，
  每条带 `type`（`boolean` / `enum` / `managedEnum`）、`options` / `coerce`、`onChange`、
  `lock`、`consentGated`。写入时按 `type` 校验并强制转换，再调 `onChange`。
  **非布尔、非枚举的项直接被拒**（"can't be set with key=value"），`managed` /
  被锁 / 需同意的项也会被拦下。`-p` 模式和 Remote Control 走同一条路径。

### 7.13 拒绝/询问规则匹配工具参数 `Tool(param:value)`（W25）

- **落点**：解析器 `chunk-rjxafr3h.js:75`（`Br(e)`）；匹配器 `chunk-qyqph83r.js:14532`（`tE`）
- **机制**：解析器把 `Agent(model:opus)` 拆成 `{toolName: "Agent", ruleContent: "model:opus"}`。
  匹配时对每条规则在 `:` 处切开——前段是参数名、后段是值，跳过工具的主字段
  （`ruleContentField`），取工具输入的对应字段（含别名映射），用 glob 匹配。
  还支持 `familyParentToolName` 向父工具递归。裸 `Agent(subagent_type)` 走另一条精确匹配路径。

### 7.14 `claude mcp login` / `logout`（W26）

- **落点**：CLI 定义 `chunk-f7m7ykk9.js:2681`（login，`--no-browser`）、`:2697`（logout）；
  处理器 `chunk-hhgy9sx3.js`；凭据存储 `chunk-c8eqqg1y.js`
- **机制**：`login` 非交互跑 OAuth，`--no-browser` 会打印授权 URL，方便 SSH / headless 环境
  把 redirect 粘回来；成功后清掉 auth 缓存并写入该 server 的凭据存储（走系统 keychain）。
  `logout` 按类型分派：`claudeai-proxy` 的凭据在 claude.ai 端（提示去网站断开）、
  `unsupported-transport` 无 OAuth、`anthropic-hosted` / `oauth` 则本地 `revokeServerTokens`。

### 7.15 shell mode 响应命令输出（W26）

- **落点**：执行 `chunk-ervcz3gn.js:225`；开关读取 `:227`
- **机制**：`! cmd` 被包成 `<bash-input>cmd</bash-input>` 的用户消息，执行完把结果
  以 `<bash-stdout>…</bash-stdout><bash-stderr>…</bash-stderr>` 追加进对话。
  关键是设置 `respondToBashCommands`（**默认 true**）：

```js
shouldQuery = respondToBashCommands && !interrupted && !backgroundTask && !aborted
```

  为真就直接触发一次模型查询，让 Claude 就着刚捕获的输出解释——**不用你再问一遍**。

### 7.16 `/rewind` 能越过 `/clear`（W26）

- **落点**：回退 `chunk-eawgw36a.js:23115`（`rewindConversationTo`）；`/clear` `chunk-zc4vwevt.js:79-229`
- **机制**：`rewindConversationTo` 作用在 transcript 快照上（`W.getSnapshot()` → 找到目标
  → `slice(0, idx)` → `W.replace(...)` 只截"工作集"）。
  而 `/clear` **不删 transcript**：它只把可见消息清空、生成新的 conversation id
  （`{type:"conversation_reset", newConversationId}`）、重置 fileHistory 快照，
  底层那份追加式会话日志留着。所以 `/rewind` 读到的快照仍含 clear 之前的消息。
- ⚠️ 这一条里"transcript 日志本身未被删除"是**强推断**：clear 的代码里只看到清工作集与
  换会话 id，没看到删日志的动作，但也没找到一处正面声明。按推断对待。

### 7.17 `/doctor` 与 `/checkup`（W28）

- **落点**：CLI `claude doctor` `chunk-f7m7ykk9.js:6005`；斜杠命令 `chunk-vyy71dp3.js:3010`
  （`name:"doctor", aliases:["checkup"]`），检查项 `:2900-3008`
- **机制**：两条路径不一样——CLI 版是**机械诊断**（安装类型、版本、keychain 探测），
  斜杠版是**提示词驱动**（`disableModelInvocation`，让 Claude 按提示词自己跑）。
  十项检查：安装健康（重复/残留安装、PATH、无法解析的 settings、坏的 frontmatter）、
  未用的 skills/MCP/plugins 占了多少上下文、CLAUDE.md 去重与冲突、裁掉可从代码库推导的内容、
  把 always-loaded 迁到 lazy skills、慢 hooks、上下文重的扩展、版本是否最新、
  建议把 auto mode 设为默认权限模式、预批频繁被拒的只读命令。
  **修复动作**都经确认门：写 `skillOverrides` / `enabledPlugins:false` 停用扩展、
  编辑或迁移 CLAUDE.md、写 `permissions.defaultMode:"auto"`、写 `permissions.allow`、建议 `claude update`。

### 7.18 Concise 输出风格（W34）

- **落点**：提示词 `chunk-qpwbvc04.js:204059`，注册表 `:204087`，turn-reminder `:204069`
- **机制**：内置风格集中在 `w8`：

```js
w8 = { default: null, Proactive, Concise, Explanatory, Learning }
```

  每项 `{name, source:"built-in", description, keepCodingInstructions, prompt, turnReminder}`。
  Concise 的 prompt 让 Claude 先给结果、不说开场白、不复述收尾。
- ⚠️ **`/output-style` 命令在这个版本已经没了**：`chunk-zx5hvy8r.js:343` 里
  `{vim:"Editor mode", "output-style":"Output style"}` 只是把它重定向到 `/config` 的对应 tab。
  风格现在由 `/config` 写入 `outputStyle`（落到 `localSettings`）。

### 7.19 `ANTHROPIC_DEFAULT_MODEL`（W34）

- **落点**：读取点 `chunk-nv0q3ger.js:13425`；`/model` 面板的来源标注 `chunk-x722nt0q.js:14542`
- **机制**：优先级链是（左高右低）：

```js
x.model || a.ANTHROPIC_MODEL || Je().model || a.ANTHROPIC_DEFAULT_MODEL
```

  即 `/model` 的运行期覆盖 > `ANTHROPIC_MODEL` > settings 的 `model` > `ANTHROPIC_DEFAULT_MODEL`。
  它**排在最低**，只在没有别的东西指定时决定新会话用哪个模型。
  `/model` 面板会把由它决定的项标成 `· Set by ANTHROPIC_DEFAULT_MODEL`。

### 7.20 `--safe-mode`（W24）

- **落点**：判定 `chunk-kn2qhfka.js:123`；状态横幅 `chunk-eawgw36a.js:8367`
- **机制**：关掉**所有自定义启动项**以排障——横幅自己列了范围：

```
Safe mode: all customizations are disabled (CLAUDE.md, skills, plugins, hooks, MCP, agents, and more)
```

  组织的 managed hooks 与 settings policy 仍然生效（但 managed plugins / skills /
  CLAUDE.md / MCP 也不生效）。同时禁用 Claude-in-Chrome 接线等。

### 7.21 `fallbackModel`（W24）

- **落点**：链构建 `chunk-hrhk4q9h.js:655`（`btr`），上限 `:672`；主循环推进 `chunk-qpwbvc04.js:178000`
- **机制**：从 `--fallbackModel`（逗号分隔）或 settings 数组读入，逐个 trim、
  `"default"` 映射、去重、校验有效性，到 3 个就停：

```js
if ((o.add(i), r.push(i), r.length === re)) break;   // var re = 3
```

  运行期按数组下标逐个尝试（`Sn[ln + 1]`），触发条件是 overloaded / API refusal /
  model_blocked，并记录 `triedModels`。另有约束：回退模型不能与主模型相同。

### 7.22 `/plugin list` 与版本要求（W23）

- **落点**：斜杠 `chunk-r9w0gtp7.js:323`；CLI `chunk-f7m7ykk9.js:2807`；
  版本区间校验 `chunk-qpwbvc04.js:67195-67237`
- **机制**：`/plugin list` 走 `pluginListHandler`，`--available` 可连 marketplace 里的
  可装插件一起列。
- **版本要求**：插件/依赖 manifest 带 `depConstraints`（依赖名 → semver 区间）。
  安装与解析时把多个区间**求交**（冲突报 disjoint / too-complex / invalid），
  再用 `satisfies(installed, range)` 校验已装版本，不满足时报
  "installed at X, which does not satisfy …"。managed 与 seed 托管的
  marketplace/插件走同一套区间机制。

---

## 八、工具与平台层

### 8.1 Monitor 工具（W15）

- **落点**：工具主体 `chunk-ac6axyj2.js:352-611`；提示词 `chunk-3e1zwnk7.js:25-85`；
  事件泵 `chunk-qpwbvc04.js:134431-134502`；工具名 `chunk-zp04wyav.js:10`
- **schema**：`command` 与 `ws` 二选一，另有 `timeout_ms`（默认 300000，上限 3600000）
  和 `persistent`（无超时，直到 TaskStop 或会话结束）。
- **机制**：`ws` 走 WebSocket，每个文本帧一个事件；`command` 走的是
  **复用 BashTool 的后台 shell 基础设施**，并在**同一个 taskRegistry** 里注册
  （`kind:"monitor"`），所以 `TaskStop` 能停它。
  事件进对话：stdout 按 200 ms 批处理、令牌桶限流（超速会显示
  `[N events suppressed]`，极端情况直接自杀），每行以 `mode:"task-notification"`、
  `priority:"next"`、包在 `<event>…</event>` 里注入会话队列。

### 8.2 Computer use（W13 / W14）

- **落点**：原生模块加载 `chunk-qpwbvc04.js:32132-32141`；会话类 `:171677-171823`；
  输入/截图执行层 `chunk-4j6ngwv0.js`；能力门控 `chunk-x722nt0q.js:22720`
- **关键结论**：**只能在 macOS 上跑**。加载 `computer_use.node` 的函数体第一行就是

```js
throw Error("@ant/computer-use-swift is only available on macOS");
```

  （`chunk-qpwbvc04.js:32136`）执行层同样在非 macOS 直接抛
  `createCliExecutor called on linux. Computer control is macOS-only.`。
  所以这个二进制里**没有** OS 级原生控制——只有加载桩、会话管理、输入/截图执行层。
- 产物里真正内联的 `computer` 工具 schema 是 **Chrome 扩展版**
  （`chunk-67f3c819.js:143`，带 `tabId`），属于 Claude in Chrome，是另一回事。

### 8.3 Windows 不再需要 Git Bash（W18）

- **落点**：回退判断 `chunk-40xnyyhy.js:352-365`；Git Bash 探测 `chunk-ye42pw2j.js:43-67`；
  PowerShell 工具 `chunk-ch4jw18f.js`
- **机制**：探测顺序是 `CLAUDE_CODE_GIT_BASH_PATH` → `C:\Program Files\Git\bin\bash.exe`
  等 → `git` 同级的 `bin`。判断逻辑很短：

```js
function us() { if (P() !== "windows") return true; return KN() !== null; }   // Bash 是否可用
// PowerShell 工具是否启用：Windows 且没有 Git Bash → 直接用
if (KN() === null) return true;
```

  有 Git Bash 时则由实验 `tengu_cobalt_ridge` 决定。
  PowerShell 工具运行时会探测是 PS7（`pwsh`）还是 Windows PowerShell 5.1，
  注入对应的 edition 提示（比如 `&&` 和三元是否可用）。

### 8.4 无闪烁 alt-screen 渲染（W14）

- **落点**：`chunk-2kd50vvr.js:16737-16771`（进出备用屏）、`:16988-17110`（帧 diff）、
  `chunk-bbpye6w7.js:113-235`（`CLAUDE_CODE_NO_FLICKER` 门控）
- **机制**：自研 Ink 风格渲染器。进备用屏写 `\x1B[?1049h` 并清屏隐藏光标；
  paint 时产出补丁列表，压成最小写操作，维护 `frontFrame/backFrame`
  **只重绘变化的行**，所以不闪。
- **开关**：`CLAUDE_CODE_NO_FLICKER=1` 强开、`CLAUDE_CODE_DISABLE_ALTERNATE_SCREEN` 关；
  tmux -CC 和"Windows over SSH（ConPTY）"下自动禁用。
  设置项里等价于 `fullscreen`。

### 8.5 一批小而具体的开关

| 特性 | 周 | 落点 | 要点 |
|---|---|---|---|
| MCP 结果大小上限（≤500K） | W14 | 常量 `chunk-zme0gc4j.js:11`（`bFe = 500000`，默认 `O7 = 50000`）；读取 `chunk-3yrq2tm2.js:3631-3689` | 服务器用工具元数据 `_meta["anthropic/maxResultSizeChars"]` 覆盖，但 `Math.min(ce, 500000)` **封顶** |
| 插件可执行文件进 PATH | W14 | `chunk-qpwbvc04.js:203740`（收集）、`:150231`（注入） | 取启用插件（非内置且有 `path`）的 `<pluginPath>/bin`，过滤含 shell 元字符的路径，然后拼进 Bash 工具的 PATH；另有 `[pluginBinaryAssets]` 按 sha256 校验后置备二进制并设可执行位 |
| 插件从 `.zip` / URL 加载 | W19 | 下载 `chunk-c5577t1e.js:1095-1156`；安装校验 `chunk-qpwbvc04.js:201307-201361`；解压 `:65311` | `--plugin-dir` 接受 `.zip`、`--plugin-url` 拉归档；下载有 256 MB / 5 跳限制，**做 sha256 完整性校验**，失败拒装；进程内解 ZIP，校验非空、提升单层 wrapper 目录、根部必须是 plugin 形状 |
| 条件 `if` 钩子 | W13 | schema `chunk-mzmfq60a.js:3490-3496`；求值 `chunk-qpwbvc04.js:224124`、`:55693` | `if` 字段的语义**就是权限规则语法**（如 `Bash(git *)`）；只在工具调用匹配时才跑。非工具事件跳过并记 "cannot be evaluated for non-tool event" |
| 钩子看到 effort 等级 | W19 | `chunk-qpwbvc04.js:39485-39501` | 组装钩子子进程环境时注入 `CLAUDE_EFFORT`；`effort.level` 也从会话状态透出。`${CLAUDE_EFFORT}` 同时是可替换占位符 |

---

## 九、跨会话消息（W32）

- **落点**：`SendMessageTool` `chunk-5m5vax5z.js`（描述 `:575-589`）；
  UDS 传输 `chunk-31rmjnrq.js`（`[uds-client]`）、`chunk-910y8j3p.js`（`[uds-messaging]`）；
  入站渲染 `chunk-181hfmv0.js`；发现工具 ListAgents `chunk-rbd8k9bq.js:93`
- **机制**：会话之间通过 **Unix domain socket** 互投消息——每个会话注册一个 inbox pipe，
  `SendMessage(to, message)` 写对端 inbox。带 pid/uid/start-token 校验，
  拒绝 stale 或非本用户的 socket，另有速率与体积上限。
  **UDS 天然不覆盖原生 Windows**，产物里多处 pull/hook 明确标 "not supported on Windows"。
- 入站消息包成 `<cross-session-message from="…">` 注入对方轮次；
  `notify_when_idle` 做一次性空闲订阅。
- **提示符里的 `@`**：`uas()` 解析输入里的 `@agent-…`，剥掉 `agent-` 前缀在已知
  agent 列表里匹配，产出 `{ type: "agent_mention", agentType }`。
  配合 ListAgents（"other Claude sessions you can message"）定位目标。

---

## 十、ultraplan / teleport / remote-control（W15 / W34）

这些共用云端骨架，放在一起看。

- **Ultraplan（W15）**：核心 `chunk-qpwbvc04.js:214784-216356`。
  用 `iT({ source:"ultraplan", permissionMode:"plan", useDefaultEnvironment:true, allowBundle:true })`
  建云会话——**首次运行 `useDefaultEnvironment:true` 自动建起云端环境**。
  之后按 phase 轮询（超时 `tengu_ultraplan_timeout_seconds` 默认 5400 s），
  phase 变成 `needs_input` 时提示去浏览器编辑器评审。
  批准后可以远端执行、在本会话实现，或者 **teleport 拉回**
  （把 `<ultraplan>` 标签里的产物注入本地会话）。
- **`/teleport`**：`chunk-bfs6prbt.js`。本地↔云端搬运会话
  （`tengu_teleport_send_to_cloud` 送上去，或拉回本地时先
  `retireConversationForTeleportPull` 清理本地 pin/artifact 状态）。
- **`/remote-env`**：`chunk-21ybnb5n.js`。列出可选环境（远端 + self-hosted runner），
  把选择写进 `userSettings.remote.defaultEnvironmentId`，供所有云会话默认落地。
- **`/remote-control`**（别名 `rc`）：`chunk-gn18jndp.js`（命令）、
  `chunk-f7m7ykk9.js:5984`（`enterRemoteControl`）。把本机/本会话桥接到 claude.ai/code
  和手机 app，需要 Trusted Device 登记，会显示 "Scan with your phone to open this session"。
  **W34 的"设备卡"这条链**在产物里是：`claude remote-control` → `enterRemoteControl`
  → `device_registry_register`（把机器登记为 device row）+ `register_device_hooks`。
  登记后这台机器在 app 里就是一张设备卡。
  这条链也是 `/autofix-pr` 的回落判断依据（"Remote Control isn't connected"）。

---

## 十一、Routines：一半在，一半不在

- **在产物里的**：本地 cron 化调度框架 `chunk-39jyx6vp.js`
  （`.claude/scheduled_tasks.json`、`tengu_scheduled_task_fire`），
  CLI 是 `claude daemon scheduled add/remove/list`（`chunk-2e4bjsz7.js`）；
  以及"本次运行本身就是一次云端 scheduled routine"时的通知适配
  `chunk-31k5d81a.js`——为真时给 PushNotification 的描述追加
  "This is a scheduled routine…"，并用 `<routine_summary>` 包裹
  （首句进手机横幅，全文进邮件正文）。
- **不在产物里的**：Web 上创建 routine、用定时/GitHub 事件/API 触发模板化云端 agent
  的那套编排。产物里只有作为被调度方的参与逻辑和文案。
- **结论**：**部分在**。别指望在这个二进制里找到 routine 的定义与触发编排。

---

## 十二、几个明确不在这个二进制里的

digest 里这些是纯 Web / Desktop / 移动端侧的，产物里**没有**实现（可能只有开关或文案）：

| 特性 | 周 | 说明 |
|---|---|---|
| self-hosted environments 的编排 | W32 | 工具名 `self_hosted_runner_*` 出现了 8 个，说明客户端侧有接口；但环境的运行与编排在组织自己的基础设施上 |
| Desktop 上的自动续跑（命中限额后自动重试） | W33 | Desktop 侧 |
| VS Code 扩展的 Focus view | W32 | 扩展侧 |
| Desktop 的 iOS Simulator 面板 | W30 | Desktop 侧 |
| Desktop 的内置浏览器 | W28 | Desktop 侧 |
| Claude Desktop on Linux | W27 | 发行侧 |
| Claude Code on the web 改版 | W17 | Web 侧 |
| mobile push notifications 的投递 | W16 | 客户端有 `PushNotification` 工具，投递在服务端 |
| Claude in Chrome 的浏览器扩展本体 | W27 | 产物里只有与之通信的工具 schema（见 8.2） |

---

## 十三、周 → 条目 对照

| 周 | 版本 | 本文档里的位置 |
|---|---|---|
| W34 | v2.1.234–239 | `/design` §4.4；Concise 输出风格 §7.18；`ANTHROPIC_DEFAULT_MODEL` §7.19；设备卡 §10 |
| W33 | v2.1.225–233 | fork 默认开 §5.1；GitLab §7.9；`@` 提会话 §9；Desktop 自动续跑 §12 |
| W32 | v2.1.220–224 | 跨会话消息 §9；self-hosted §12；auto mode 默认 §2.5；VS Code §12 |
| W30 | v2.1.214–219 | Opus 5 §6.1；`/code-review` 改后台子代理 §3.1；Security 插件；iOS Simulator §12 |
| W29 | v2.1.207–212 | Artifacts 调 connector §4.2；分享/角色 §4.3；屏幕阅读器 §7.5；`/fork` §5.2 |
| W28 | v2.1.202–206 | `/doctor` §7.17；auto mode 拦 transcript 篡改与 `rm -rf "$VAR"` §2.4；agent view 状态词 §5.6 |
| W27 | v2.1.195–201 | Sonnet 5 §6.1；Claude in Chrome §8.2；子代理默认后台 §5.4；`/radio` §7.7 |
| W26 | v2.1.185–193 | `claude mcp login/logout` §7.14；shell mode §7.15；`/rewind` 越过 `/clear` §7.16；权限弹窗回传 §5.4 |
| W25 | v2.1.178–183 | Artifacts §4.1；`Tool(param:value)` §7.13；`/config key=value` §7.12；破坏性 git 规则 §2.4 |
| W24 | v2.1.166–176 | `/cd` §7.11（+缓存原理 §6.5）；嵌套深度 §5.3；`--safe-mode` §7.20；`fallbackModel` §7.21 |
| W23 | v2.1.158–165 | 3P 上的 auto mode §2.5；自动编辑更安全；`/plugin list` 与版本要求 §7.22 |
| W22 | v2.1.150–157 | Opus 4.8 §6.1；dynamic workflows §5.5；security-guidance 插件；fast mode §6.3 |
| W21 | v2.1.143–149 | `/code-review` §3.1；`/usage` 拆解 §7.6；后台会话进 `/resume` §5.6 |
| W20 | v2.1.139–142 | agent view §5.6；`/goal` §7.1；fast mode §6.3；Rewind 压缩上下文 |
| W19 | v2.1.128–136 | 插件 `.zip`/URL §8.5；`worktree.baseRef` §7.9；hard deny §2.4；钩子看 effort §8.5 |
| W18 | v2.1.120–126 | Windows 免 Git Bash §8.3；`claude ultrareview` §3.2；`claude project purge`；PR URL 进 `/resume` |
| W17 | v2.1.114–119 | `/ultrareview` §3.2；session recap §7.3；自定义主题 §7.4；Web 改版 §12 |
| W16 | v2.1.105–113 | Opus 4.7 §6.1；Routines §11；推送；`/usage` §7.6 |
| W15 | v2.1.92–101 | Ultraplan §10；Monitor §8.1；`/loop` 自调速 §7.2；`/team-onboarding` §7.10 |
| W14 | v2.1.86–91 | computer use §8.2；`/powerup` §7.8；alt-screen §8.4；MCP 上限 §8.5；插件 PATH §8.5 |
| W13 | v2.1.83–85 | auto mode 本体 §2；computer use §8.2；`/autofix-pr` §3.3；条件 `if` 钩子 §8.5 |

digest 里没进这张表的少数几条（`claude project purge`、Rewind 的"Summarize up to here"、
security-guidance 插件、预批只读命令等），要么是插件本体（不在这个二进制里），
要么尚未单独核查——**不写进表格，也不假装覆盖了**。

---

## 十四、这份文档的可信度

- 上面每一条的落点都是**实际读代码**得到的，没有一条靠文件名或命名习惯猜。
- 关键结论我都独立复核过一遍（原文片段在"证据"栏里，可以直接回去核）。
- 有 **3 处**调查过程中纠正了原始假设，我保留了纠正后的说法并标注了分歧：
  1. **auto mode 没有本地分类器模型**——是一次普通 API 调用加一份提示词资产（§2）。
  2. **子代理嵌套深度产物里是 3 不是 5**——那个 5 由服务端下发（§5.3）。
  3. **`/output-style` 命令已被移除**，只剩重定向（§7.18）。
- 有 **2 处**证据不足，已就地标注，没有拔高成结论：
  - `/rewind` 越过 `/clear` 的"transcript 日志未被删除"是强推断（§7.16）。
  - "后台子代理此前自动拒绝权限"这条**变更前**的行为在产物里找不到（§5.4）——
    代码里只有现在的共享 `canUseTool` 通路，历史状态自然不在代码里。

---

## 十五、版权

代码是 Anthropic PBC 的闭源产物，每个模块头部都带有其版权与许可声明。
本文档只作本地逆向分析与互操作性研究之用，不要再分发。
