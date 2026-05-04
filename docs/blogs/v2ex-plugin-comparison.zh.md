# 锐评给 Claude Code 和 Codex 开发插件的体验

前阵子给 [Chorus](https://github.com/Chorus-AIDLC/Chorus) 写 CLI 插件,让 coding agent 可以自己 drive 从设计到交付的流程。分别给 Claude Code 和 Codex CLI 上各写了一版,开发体验差距巨大。

基准版本: Claude Code `2.1.126`，Codex CLI `0.128.0`。两边都在快速迭代,后面版本应该会修掉其中一些问题。

完整对比和踩坑记录我写了一篇长文放在博客了: https://chorus-ai.dev/zh/blog/claude-code-vs-codex-plugin-systems/ V2EX 这里挑几个印象最深的聊聊。

---

**省流**：最后有具体的模块打分对比

## 一、装完能不能直接跑

Chorus 本体是一个 Web 服务,agent 通过 MCP 工具跟它交互。插件的职责之一是把 MCP server 配进 Agent,让用户不必自己去配置 MCP。为了达到最佳的使用体验，肯定是用户操作的步骤越少越好。

Claude Code 那边用户输两条 slash command 就完事,[`.mcp.json`](https://github.com/Chorus-AIDLC/Chorus/blob/main/public/chorus-plugin/.mcp.json) 声明在插件里,`${VAR}` 运行时展开:

```
/plugin marketplace add Chorus-AIDLC/Chorus
/plugin install chorus@chorus-plugins
```
这样插件自带的 MCP 就能跑在任意一个项目里了，给到**顶级**

Codex 这边就困难多了。第一,虽然有命令可以从 marketplace 安装插件，但是启用还需要用户手动在 TUI 里操作没办法复制一条命令直接启用。第二,配置里所有几乎字段都是字面量,不做 `${VAR}` 环境变量展开，对于不支持环境变量的字段比如 MCP 的 URL，需要通过脚本写入到 Codex 配置文件里。

最终我写了个 [Bash installer](https://github.com/Chorus-AIDLC/Chorus/blob/main/public/install-codex.sh) 帮助用户改配置文件硬凑出"一条命令装好"的体感, 但脚本脆得很,Codex 做一些配置层面的重构就得重写。给到**NPC**

## 二、钩子能不能跟着插件走

插件比起纯用 MCP + Skill 来说最大的好处就是 Hooks，为了保证 Agent 能按照预期跑任务，需要在各种生命周期钩子中埋脚本。Chorus 最基本的功能需要三个钩子: session 启动时调 `chorus_checkin` 把当前 agent 的身份和待办注入上下文;agent 提交方案以及任务后分别触发独立的 Reviewer Agent 对抗检查。Claude Code 和 Codex 都支持 Hooks，但这方面的差距也是最大的。

Claude Code 那边就是照着写。插件里放一份 [`hooks/hooks.json`](https://github.com/Chorus-AIDLC/Chorus/blob/main/public/chorus-plugin/hooks/hooks.json),用 `${CLAUDE_PLUGIN_ROOT}` 指向插件脚本,harness 自动注册。所有的事情都符合预期，你在插件中定义的生命周期钩子会自动在对应的事件发生时触发。**夯爆了**

Codex 这边我也是这么做的: 插件里放 `hooks.json`,manifest 里 `hooks` 字段指过去,官方 example 里也是这么写的。装完看 `/plugins` 面板钩子显示已就位,但 session 启动时钩子根本不跑。

我一开始以为自己哪里写错了,换了好几种写法、改 matcher、reinstall、对着 example 逐字比对,大半天就这么过去了。最后去仓库搜 issue 才找到 [#16430](https://github.com/openai/codex/issues/16430): **plugin manifest 解析器只认 `skills` / `mcpServers` / `apps`,不认 `hooks`**,钩子发现逻辑只扫 config layer 下的 `hooks.json`,不扫已安装插件的根目录，这个功能压根就对第三方插件不工作。截至目前版本，Codex 虽然文档里口口声声说插件支持这个功能，但是源码里居然是一个大大的 TODO，离了大谱。**拉完了**

## 三、钩子事件够不够用

Chorus 想通过插件实现的另一个一个功能是多 agent 并行工作时的可观测性: 用户起 5 个子代理并行写代码,要在看板上看到谁在做哪个任务、做到哪一步、有没有心跳。完全靠 Agent 自己调 MCP 上报不仅费 Token 还不靠谱，用生命周期钩子完成非常合适。

Claude Code 的钩子事件覆盖了子 Agent 整个生命周期: `SubagentStart`、`SubagentStop`、`TeammateIdle`、`TaskCompleted`、`SessionEnd`,子代理一起来 harness 就能自动建 session、发心跳、关 session。给到**顶级**，不给夯是因为一些很小的细节，比如无法在 Subagent 关闭后在钩子处注入上下文给主 Agent 等。虽然看起来很小但在多 Agent 协作的时候加工一下 Agent 之间交流的信息对控制行为非常重要。

Codex 这边只有 6 个钩子: `SessionStart`、`UserPromptSubmit`、`PreToolUse`、`PostToolUse`、`PermissionRequest`、`Stop`。**整个 agent 生命周期相关的事件都没有**。主 Agent 调 `spawn_agent` 之后,子代理的存在对插件完全不可见,只能让主代理自己在 prompt 里记得"spawn 前调这个、spawn 后调那个",LLM 大部分时候能遵守,偶尔漏一步就是一个泄漏的 session 挂在那里。大的钩子都有，小钩子缺很多，给到**NPC**吧

## 四、子代理能不能当一等公民

Chorus 有两个 reviewer agent: proposal 提交后自动跑 proposal-reviewer 审方案,task 提交后自动跑 task-reviewer 审代码。这俩 agent 必须是**只读**的(不能 Edit、Write、Bash),不然让它们改了代码就不叫 review 了。权限这事儿最好 harness 在调用层就管住,靠 LLM 自觉不太行。并且最好还能控制最大轮数，防止在独立的 Review Agent 中陷入死循环或者跑偏影响主要任务的推进。

Claude Code 上就是一份 [`agents/proposal-reviewer.md`](https://github.com/Chorus-AIDLC/Chorus/blob/main/public/chorus-plugin/agents/proposal-reviewer.md):

```yaml
---
description: "Review submitted Chorus proposals for quality"
model: inherit
maxTurns: 20
disallowedTools: [Agent, Edit, Write, NotebookEdit, Bash]
---
```

可以看到在 Agent 定义的 meta 信息中支持多种配置，允许使用特定模型，支持限制最大轮次，允许屏蔽各种工具的使用等等。

文件正文就是 reviewer 的 system prompt,主代理 `Task(subagent_type: "chorus:proposal-reviewer")` 一下就起来了,工具权限、模型、轮次上限全都在 harness 层强制。**夯爆了！**

Codex 的 `spawn_agent` 工具只认四个内置 role: `default` / `explorer` / `worker` / `awaiter`,**插件 manifest 里没有注册新 role 的字段**。我一开始还以为 skill 目录下那个 `agents/openai.yaml` 能注册,实测 `spawn_agent(agent_type="chorus-proposal-reviewer")` 直接报 `unknown agent_type`,翻完 Rust 源码才确认 `openai.yaml` 只是给 TUI 面板显示用的元数据，纯看看而已。

最后发现了自定义子 Agent 的方式: 把 reviewer 当成 [skill](https://github.com/Chorus-AIDLC/Chorus/blob/main/plugins/chorus/skills/chorus-proposal-reviewer/SKILL.md),用内置的 `default` role spawn,通过 `items` 数组把 skill 塞进去:

```
spawn_agent(
  agent_type="default",
  items=[
    { type: "skill", path: "chorus:chorus-proposal-reviewer" },
    { type: "text",  text: "Review proposal <uuid>. Post VERDICT." }
  ]
)
```

能跑,但因为是 `default` role,子代理什么工具都能用,只能在 SKILL.md 里硬写"严禁修改任何文件、严禁运行 Bash，最高不要调用 20 词工具"，靠 LLM 自觉，太玄乎了。给到**顶级**，虽然不支持元数据，但是 Skill as Agent 的设计还是挺好玩的

还有个暗雷: Codex 每条根线程最多 6 个并发子代理,`completed` 状态**不释放槽位**,必须显式 `close_agent(id)`,第 7 次一定爆 `agent thread limit reached`，最好在 skill 里提醒一下主 Agent。

## 五、文档和调试

这一条写得最有感触,因为前面几节那种"按文档写了但就是不工作"的坑,逼得我必须有办法自己核实真相。

Codex 这边有个意外的好处: **代码完全开源**。文档不齐没关系,翻 `codex-rs/` 下的 Rust 源码就能把行为坐实。我后面一堆结论(`spawn_agent` 只接受四个内置 role、`config.toml` 不展开变量、`completed` 子代理不释放线程槽位)都是靠读源码才敢下断言的。文档跟不上进度,但真相至少是可读的。

Claude Code 正好反过来: 文档写得齐整,代码却是黑盒。钩子事件字段、MCP 变量展开规则都交代清楚,但文档没覆盖到的行为就只能靠猜。好在前阵子 Claude Code 有一次"开源"的契机,社区里能找到比较完整的源码,很多之前只能猜的细节现在能验证了。（不过很多新的功能没有，得等下一次开源。。）

开源和文档两件事上两边各赢一半: Claude Code 文档整齐,Codex 源码随时可读,实际做插件的时候这两种资源的价值互补(要不然 Claude Code 你直接开源算了)。

## 打分
我在原文对九个维度进行了打分，大家也可以参考下

| 维度 | Claude Code | Codex |
|---|---:|---:|
| 安装 | 5 | 2 |
| MCP 集成 | 5 | 1 |
| 钩子交付 | 5 | 0 |
| 钩子事件覆盖 | 5 | 2 |
| 子代理 | 5 | 2 |
| Skills | 5 | 4 |
| 配置变量 | 5 | 1 |
| Marketplace | 4 | 3 |
| 文档和调试 | 4 | 4 |
| **合计** | **43** | **19** |

数字看着悬殊,但不是说 Codex 不能用,只是你要付出额外的很多精力来做 Harness Engineering 来保证你的插件能正确指引 Agent 干活。

---

结论就是: 如果你想写**纯 skill 集合**,两边差不多，如果想和 Chorus 一样做一些流程控制等精细的操作，那么 Codex 这边就欠缺太多基础建设了。没有对比就没有伤害，虽然最近 A\ 各种封杀降智不做人，把口碑都败光了，但从 Claude Code 的插件扩展性来看在 Harness Engineering 的理解上确实是断档领先，很多我们习以为常的功能其实是花了大力气去打磨的。也难怪 Claude Code 的生态那么有活力，希望 Codex 赶快抄起来吧。

